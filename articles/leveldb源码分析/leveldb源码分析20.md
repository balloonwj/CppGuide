# leveldb源码分析20

## 本系列《leveldb源码分析》共有22篇文章，这是第二十篇



### 12 DB的打开

先分析LevelDB是如何打开db的，万物始于创建。在打开流程中有几个辅助函数：**DBImpl()，DBImpl::Recover, DBImpl::DeleteObsoleteFiles, DBImpl::RecoverLogFile, DBImpl::MaybeScheduleCompaction**。

### 12.1 DB::Open()

打开一个db，进行PUT、GET操作，就是前面的静态函数**DB::Open**的工作。如果操作成功，它就返回一个db指针。前面说过DB就是一个接口类，其具体实现在DBImp类中，这是一个DB的子类。
函数声明为：

```
Status DB::Open(const Options& options, const std::string&dbname, DB** dbptr);
```

分解来看，Open()函数主要有以下5个执行步骤。
S1 创建DBImpl对象，其后进入**DBImpl::Recover()**函数执行S2和S3。
S2 从已存在的db文件恢复db数据，根据CURRENT记录的MANIFEST文件读取db元信息；这通过调用**VersionSet::Recover()**完成。
S3 然后过滤出那些最近的更新log，前一个版本可能新加了这些log，但并没有记录在MANIFEST中。然后依次根据时间顺序，调用**DBImpl::RecoverLogFile()**从旧到新回放这些操作log。回放log时可能会修改db元信息，比如dump了新的level 0文件，因此它将返回一个VersionEdit对象，记录db元信息的变动。
S4 如果**DBImpl::Recover()**返回成功，就执行**VersionSet::LogAndApply()**应用VersionEdit，并保存当前的DB信息到新的MANIFEST文件中。
S5 最后删除一些过期文件，并检查是否需要执行compaction，如果需要，就启动后台线程执行。
下面就来具体分析Open函数的代码，在Open函数中涉及到上面的3个流程。
S1 首先创建DBImpl对象，锁定并试图做Recover操作。Recover操作用来处理创建flag，比如存在就返回失败等等，尝试从已存在的sstable文件恢复db。并返回db元信息的变动信息，一个VersionEdit对象。

```
DBImpl* impl = newDBImpl(options, dbname);  
impl->mutex_.Lock(); // 锁db  
VersionEdit edit;  
Status s =impl->Recover(&edit); // 处理flag&恢复：create_if_missing,error_if_exists 
```

S2 如果Recover返回成功，则调用VersionSet取得新的log文件编号——实际上是在当前基础上+1，准备新的log文件。如果log文件创建成功，则根据log文件创建log::Writer。然后执行VersionSet::LogAndApply，根据edit记录的增量变动生成新的current version，并写入MANIFEST文件。

函数**NewFileNumber(){returnnext_file_number_++;}**，直接返回**next_file_number_**。

```
 uint64_t new_log_number = impl->versions_->NewFileNumber();
 WritableFile* lfile;
 s = options.env->NewWritableFile(LogFileName(dbname, new_log_number), &lfile);
 if (s.ok()) {
    edit.SetLogNumber(new_log_number);
    impl->logfile_ = lfile;
    impl->logfile_number_ = new_log_number;
    impl->log_ = newlog::Writer(lfile);
    s = impl->versions_->LogAndApply(&edit, &impl->mutex_);
}
```

S3 如果VersionSet::LogAndApply返回成功，则删除过期文件，检查是否需要执行compaction，最终返回创建的DBImpl对象。

```
if (s.ok()) {
    impl->DeleteObsoleteFiles();
    impl->MaybeScheduleCompaction();
}
impl->mutex_.Unlock();
if (s.ok()) *dbptr = impl;
return s;
```

以上就是DB::Open的主题逻辑。

**12.2 DBImpl::DBImpl()**

构造函数做的都是初始化操作，

```
DBImpl::DBImpl(const Options& options, const std::string&dbname)
```

首先是初始化列表中，直接根据参数赋值，或者直接初始化。Comparator和filter policy都是参数传入的。在传递option时会首先将option中的参数合法化，**logfile_number_**初始化为0，指针初始化为NULL。
创建MemTable，并增加引用计数，创建WriteBatch。

```
mem_(newMemTable(internal_comparator_)),
tmp_batch_(new WriteBatch),
mem_->Ref();
// 然后在函数体中，创建TableCache和VersionSet。  
// 为其他预留10个文件，其余的都给TableCache.  
const int table_cache_size = options.max_open_files - 10;
table_cache_ = newTableCache(dbname_, &options_, table_cache_size);
versions_ = newVersionSet(dbname_, &options_, table_cache_, &internal_comparator_);
```

###  

### 12.3 DBImp::NewDB()

当外部在调用DB::Open()时设置了option指定如果db不存在就创建，如果db不存在leveldb就会调用函数创建新的db。判断db是否存在的依据是**<db name>/CURRENT**文件是否存在。其逻辑很简单。

```
// S1首先生产DB元信息，设置comparator名，以及log文件编号、文件编号，以及seq no。  
VersionEdit new_db;
new_db.SetComparatorName(user_comparator()->Name());
new_db.SetLogNumber(0);
new_db.SetNextFile(2);
new_db.SetLastSequence(0);
// S2 生产MANIFEST文件，将db元信息写入MANIFEST文件。  
const std::string manifest = DescriptorFileName(dbname_, 1);
WritableFile* file;
Status s = env_->NewWritableFile(manifest, &file);
if (!s.ok()) return s;
{
    log::Writer log(file);
    std::string record;
    new_db.EncodeTo(&record);
    s = log.AddRecord(record);
    if (s.ok()) s = file->Close();
}
delete file;
// S3 如果成功，就把MANIFEST文件名写入到CURRENT文件中  
if (s.ok()) s = SetCurrentFile(env_, dbname_, 1);
else env_->DeleteFile(manifest);
return s;
```

这就是创建新DB的逻辑，很简单。

**12.4 DBImpl::Recover()**

函数声明为：

```
StatusDBImpl::Recover(VersionEdit* edit)
```

如果调用成功则设置VersionEdit。Recover的基本功能是：首先是处理创建flag，比如存在就返回失败等等；然后是尝试从已存在的sstable文件恢复db；最后如果发现有大于原信息记录的log编号的log文件，则需要回放log，更新db数据。回放期间db可能会dump新的level 0文件，因此需要把db元信息的变动记录到edit中返回。函数逻辑如下：

S1 创建目录，目录以db name命名，忽略任何创建错误，然后尝试获取d**b name/LOCK**文件锁，失败则返回。

```
env_->CreateDir(dbname_);
Status s = env_->LockFile(LockFileName(dbname_), &db_lock_);
if (!s.ok()) return s;
```

S2 根据CURRENT文件是否存在，以及option参数执行检查。
如果文件不存在**&create_is_missing=true**，则调用函数NewDB()创建；否则报错。
如果文件存在& error_if_exists=true，则报错。
S3 调用VersionSet的**Recover()**函数，就是从文件中恢复数据。如果出错则打开失败，成功则向下执行S4。

```
s = versions_->Recover();
```

S4尝试从所有比manifest文件中记录的log要新的log文件中恢复（前一个版本可能会添加新的log文件，却没有记录在manifest中）。另外，函数PrevLogNumber()已经不再用了，仅为了兼容老版本。

```
//  S4.1 这里先找出所有满足条件的log文件：比manifest文件记录的log编号更新。  
SequenceNumber max_sequence(0);
const uint64_t min_log = versions_->LogNumber();
const uint64_t prev_log = versions_->PrevLogNumber();
std::vector<std::string>filenames;
s = env_->GetChildren(dbname_, &filenames); // 列出目录内的所有文件  
uint64_t number;
FileType type;
std::vector<uint64_t>logs;
for (size_t i = 0; i < filenames.size(); i++) { // 检查log文件是否比min log更新  
    if (ParseFileName(filenames[i], &number, &type) && type == kLogFile
        && ((number >= min_log) || (number == prev_log))) {
        logs.push_back(number);
    }
}
//  S4.2 找到log文件后，首先排序，保证按照生成顺序，依次回放log。并把DB元信息的变动（sstable文件的变动）追加到edit中返回。  
std::sort(logs.begin(), logs.end());
for (size_t i = 0; i < logs.size(); i++) {
    s = RecoverLogFile(logs[i], edit, &max_sequence);
    // 前一版可能在生成该log编号后没有记录在MANIFEST中，  
    //所以这里我们手动更新VersionSet中的文件编号计数器  
    versions_->MarkFileNumberUsed(logs[i]);
}
//  S4.3 更新VersionSet的sequence  
if (s.ok()) {
    if (versions_->LastSequence() < max_sequence)
        versions_->SetLastSequence(max_sequence);
}
```

上面就是Recover的执行流程。

### 12.5 DBImpl::DeleteObsoleteFiles()

这个是垃圾回收函数，如前所述，每次compaction和recovery之后都会有文件被废弃。DeleteObsoleteFiles就是删除这些垃圾文件的，它在每次compaction和recovery完成之后被调用。
其调用点包括：**DBImpl::CompactMemTable,DBImpl::BackgroundCompaction,** 以及DB::Open的**recovery**步骤之后。
它会删除所有过期的log文件，没有被任何level引用到、或不是正在执行的compaction的output的sstable文件。
该函数没有参数，其代码逻辑也很直观，就是列出db的所有文件，对不同类型的文件分别判断，如果是过期文件，就删除之，如下：

```
// S1 首先，确保不会删除pending文件，将versionset正在使用的所有文件加入到live中。  
std::set<uint64_t> live = pending_outputs_;
versions_->AddLiveFiles(&live); //该函数其后分析  
                                // S2 列举db的所有文件  
std::vector<std::string>filenames;
env_->GetChildren(dbname_, &filenames);
// S3 遍历所有列举的文件，根据文件类型，分别处理；  
uint64_t number;
FileType type;
for (size_t i = 0; i < filenames.size(); i++) {
    if (ParseFileName(filenames[i], &number, &type)) {
        bool keep = true; //false表明是过期文件  
                          // S3.1 kLogFile，log文件，根据log编号判断是否过期  
        keep = ((number >= versions_->LogNumber()) ||
            (number == versions_->PrevLogNumber()));
        // S3.2 kDescriptorFile，MANIFEST文件，根据versionset记录的编号判断  
        keep = (number >= versions_->ManifestFileNumber());
        // S3.3 kTableFile，sstable文件，只要在live中就不能删除  
        // S3.4 kTempFile，如果是正在写的文件，只要在live中就不能删除  
        keep = (live.find(number) != live.end());
        // S3.5 kCurrentFile,kDBLockFile, kInfoLogFile，不能删除  
        keep = true;
        // S3.6 如果keep为false，表明需要删除文件，如果是table还要从cache中删除  
        if (!keep) {
            if (type == kTableFile) table_cache_->Evict(number);
            Log(options_.info_log, "Delete type=%d #%lld\n", type, number);
            env_->DeleteFile(dbname_ + "/" + filenames[i]);
        }
    }
}
```

这就是删除过期文件的逻辑，其中调用到了**VersionSet::AddLiveFiles**函数，保证不会删除active的文件。

函数DbImpl::MaybeScheduleCompaction()放在Compaction一节分析，基本逻辑就是如果需要compaction，就启动后台线程执行compaction操作。

### 12.6 DBImpl::RecoverLogFile()

函数声明：

```
StatusRecoverLogFile(uint64_t log_number, VersionEdit* edit,SequenceNumber* max_sequence)
```

参数说明：
@log_number是指定的log文件编号
@edit记录db元信息的变化——sstable文件变动
@max_sequence 返回max{log记录的最大序号, *max_sequence}
该函数打开指定的log文件，回放日志。期间可能会执行compaction，生产新的level 0sstable文件，记录文件变动到edit中。
它声明了一个局部类LogReporter以打印错误日志，没什么好说的，下面来看代码逻辑。

```
// S1 打开log文件返回SequentialFile*file，出错就返回，否则向下执行S2。  
// S2 根据log文件句柄file创建log::Reader，准备读取log。  
log::Reader reader(file, &reporter, true/*checksum*/, 0/*initial_offset*/);
// S3 依次读取所有的log记录，并插入到新生成的memtable中。这里使用到了批量更新接口WriteBatch，具体后面再分析。  
std::string scratch;
Slice record;
WriteBatch batch;
MemTable* mem = NULL;
while (reader.ReadRecord(&record, &scratch) && status.ok()) { // 读取全部log  
    if (record.size() < 12) { // log数据错误，不满足最小长度12  
        reporter.Corruption(record.size(), Status::Corruption("log recordtoo small"));
        continue;
    }
    WriteBatchInternal::SetContents(&batch, record); // log内容设置到WriteBatch中  
    if (mem == NULL) { // 创建memtable  
        mem = new MemTable(internal_comparator_);
        mem->Ref();
    }
    status = WriteBatchInternal::InsertInto(&batch, mem); // 插入到memtable中  
    MaybeIgnoreError(&status);
    if (!status.ok()) break;
    const SequenceNumber last_seq =
        WriteBatchInternal::Sequence(&batch) + WriteBatchInternal::Count(&batch) - 1;
    if (last_seq > *max_sequence) *max_sequence = last_seq; // 更新max sequence  
                                                           // 如果mem的内存超过设置值，则执行compaction，如果compaction出错，
                                                           // 立刻返回错误，DB::Open失败  
    if (mem->ApproximateMemoryUsage() > options_.write_buffer_size) {
        status = WriteLevel0Table(mem, edit, NULL);
        if (!status.ok()) break;
        mem->Unref(); // 释放当前memtable  
        mem = NULL;
    }
}
// S4 扫尾工作，如果mem != NULL，说明还需要dump到新的sstable文件中。  
if (status.ok() && mem != NULL) {// 如果compaction出错，立刻返回错误  
    status = WriteLevel0Table(mem, edit, NULL);
}
if (mem != NULL)mem->Unref();
delete file;
return status;
```

把MemTabledump到sstable是函数WriteLevel0Table的工作，其实这是compaction的一部分，准备放在compaction一节来分析。

**12.7 小结**

如上DB打开的逻辑就已经分析完了，打开逻辑参见DB::Open()中描述的5个步骤。此外还有两个东东：把Memtable dump到sstable的WriteLevel0Table()函数，以及批量修改WriteBatch。第一个放在后面的compaction一节，第二个放在DB更新操作中。接下来就是db的关闭。

## 

### 13 DB的关闭&销毁

### 13.1 DB关闭

外部调用者通过DB::Open()获取一个DB*对象，如果要关闭打开的DB*db对象，则直接delete db即可，这会调用到DBImpl的析构函数。
析构依次执行如下的5个逻辑：
S1 等待后台compaction任务结束
S2 释放db文件锁，<dbname>/lock文件
S3 删除VersionSet对象，并释放MemTable对象
S4 删除log相关以及TableCache对象
S5 删除options的block_cache以及info_log对象

### 13.2 DB销毁

函数声明：

```
StatusDestroyDB(const std::string& dbname, const Options& options)
```

该函数会删除掉db的数据内容，要谨慎使用。函数逻辑为：
S1 获取dbname目录的文件列表到filenames中，如果为空则直接返回，否则进入S2。
S2 锁文件<dbname>/lock，如果锁成功就执行S3
S3 遍历filenames文件列表，过滤掉lock文件，依次调用DeleteFile删除。
S4 释放lock文件，并删除之，然后删除文件夹。
Destory就执行完了，如果删除文件出现错误，记录之，依然继续删除下一个。最后返回错误代码。
看来这一章很短小。DB的打开关闭分析完毕。