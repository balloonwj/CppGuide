# leveldb源码分析19

本系列《leveldb源码分析》共有22篇文章，这是第十九篇

## 11.VersionSet分析之2



### 11.4 LogAndApply()

函数声明：

```
Status LogAndApply(VersionEdit*edit, port::Mutex* mu)
```

前面接口小节中讲过其功能：在currentversion上应用指定的VersionEdit，生成新的**MANIFEST**信息，保存到磁盘上，并用作**current version**，故为Log And Apply。
参数edit也会被函数修改。



#### 11.4.1 函数流程

下面就来具体分析函数代码。
**S1 为edit设置log number等4个计数器。**

```
if (edit->has_log_number_) {
    assert(edit->log_number_ >= log_number_);
    assert(edit->log_number_ < next_file_number_);
}
else edit->SetLogNumber(log_number_);
if (!edit->has_prev_log_number_) edit->SetPrevLogNumber(prev_log_number_);
edit->SetNextFile(next_file_number_);
edit->SetLastSequence(last_sequence_);
```

要保证edit自己的log number是比较大的那个，否则就是致命错误。保证edit的log number小于next file number，否则就是致命错误-见9.1小节。 

**S2 创建一个新的Version v，并把新的edit变动保存到v中。** 

```
Version* v = new Version(this);
{
    Builder builder(this, current_);
    builder.Apply(edit);
    builder.SaveTo(v);
}
Finalize(v); //如前分析，只是为v计算执行compaction的最佳level  
```

**S3 如果MANIFEST文件指针不存在，就创建并初始化一个新的MANIFEST文件。这只会发生在第一次打开数据库时。这个MANIFEST文件保存了current version的快照。**

```
std::string new_manifest_file;
Status s;
if (descriptor_log_ == NULL) {
    // 这里不需要unlock *mu因为我们只会在第一次调用LogAndApply时  
    // 才走到这里(打开数据库时).  
    assert(descriptor_file_ == NULL); // 文件指针和log::Writer都应该是NULL  
    new_manifest_file = DescriptorFileName(dbname_, manifest_file_number_);
    edit->SetNextFile(next_file_number_);
    s = env_->NewWritableFile(new_manifest_file, &descriptor_file_);
    if (s.ok()) {
        descriptor_log_ = new log::Writer(descriptor_file_);
        s = WriteSnapshot(descriptor_log_); // 写入快照  
    }
}
```

**S4 向MANIFEST写入一条新的log，记录current version的信息。在文件写操作时unlock锁，写入完成后，再重新lock，以防止浪费在长时间的IO操作上。**

```
[cpp] view plain copy
mu->Unlock();
if (s.ok()) {
    std::string record;
    edit->EncodeTo(&record);// 序列化current version信息  
    s = descriptor_log_->AddRecord(record); // append到MANIFEST log中  
    if (s.ok()) s = descriptor_file_->Sync();
    if (!s.ok()) {
        Log(options_->info_log, "MANIFEST write: %s\n", s.ToString().c_str());
        if (ManifestContains(record)) { // 返回出错，其实确实写成功了  
            Log(options_->info_log, "MANIFEST contains log record despiteerror ");
            s = Status::OK();
        }
    }
}
//如果刚才创建了一个MANIFEST文件，通过写一个指向它的CURRENT文件  
//安装它；不需要再次检查MANIFEST是否出错，因为如果出错后面会删除它  
if (s.ok() && !new_manifest_file.empty()) {
    s = SetCurrentFile(env_, dbname_, manifest_file_number_);
}
mu->Lock();
```

**S5 安装这个新的version**

```
if (s.ok()) { // 安装这个version  
    AppendVersion(v);
    log_number_ = edit->log_number_;
    prev_log_number_ = edit->prev_log_number_;
}
else { // 失败了，删除  
    delete v;
    if (!new_manifest_file.empty()) {
        delete descriptor_log_;
        delete descriptor_file_;
        descriptor_log_ = descriptor_file_ = NULL;
        env_->DeleteFile(new_manifest_file);
    }
}
```

流程的S4中，函数会检查MANIFEST文件是否已经有了这条record，那么什么时候会有呢？

主函数使用到了几个新的辅助函数WriteSnapshot，ManifestContains和SetCurrentFile，下面就来分析。

#### 11.4.2 WriteSnapshot()

函数声明：

```
Status WriteSnapshot(log::Writer*log)
```

把currentversion保存到*log中，信息包括comparator名字、compaction点和各级sstable文件，函数逻辑很直观。

- S1 首先声明一个新的**VersionEdit edit**；
- S2 设置comparator：**edit.SetComparatorName(icmp_.user_comparator()->Name())**;
- S3 遍历所有level，根据compact_pointer_[level]，设置compaction点：
    **edit.SetCompactPointer(level, key)**;
- S4 遍历所有level，根据current_->files_，设置sstable文件集合：**edit.AddFile(level, xxx)**
- S5 根据序列化并append到log（MANIFEST文件）中；

```
std::string record;
edit.EncodeTo(&record);
returnlog->AddRecord(record);
```



以上就是WriteSnapshot的代码逻辑。



#### 11.4.3 ManifestContains()

函数声明：

```
bool ManifestContains(conststd::string& record)
```

如果当前MANIFEST包含指定的record就返回true，来看看函数逻辑。

- S1 根据当前的manifest_file_number_文件编号打开文件，创建**SequentialFile**对象

- S2 根据创建的SequentialFile对象创建**log::Reader**，以读取文件

- S3 调用log::Reader的ReadRecord依次读取record，如果和指定的record相同，就返回true，没有相同的record就返回false



**SetCurrentFile**很简单，就是根据指定manifest文件编号，构造出MANIFEST文件名，并写入到CURRENT即可。



### 11.5 ApproximateOffsetOf()

函数声明：

```
uint64_tApproximateOffsetOf(Version* v, const InternalKey& ikey)
```

在指定的version中查找指定key的大概位置。
假设version中有n个**sstable**文件，并且落在了地i个sstable的key空间内，那么返回的位置**= sstable1文件大小+sstable2文件大小 + … + sstable (i-1)文件大小
\+ key在sstable i中的大概偏移**。
可分为两段逻辑。

- 首先直接和sstable的max key作比较，如果key > max key，直接跳过该文件，还记得sstable文件是有序排列的。
    对于level >0的文件集合而言，如果如果key < sstable文件的min key，则直接跳出循环，因为后续的sstable的min key肯定大于key。

    

- key在sstable i中的大概偏移使用的是Table:: ApproximateOffsetOf(target)接口，前面分析过，它返回的是Table中>= target的key的位置。



**VersionSet的相关函数**暂时分析到这里，compaction部分后需单独分析。
