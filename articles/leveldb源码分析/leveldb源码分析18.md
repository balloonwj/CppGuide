# leveldb源码分析18

本系列《leveldb源码分析》共有22篇文章，这是第十八篇

**11 VersionSet分析之1**
`Version`之后就是`VersionSet`，它并不是Version的简单集合，还肩负了不少的处理逻辑。这里的分析不涉及到`compaction`相关的部分，这部分会单独分析。包括log等各种编号计数器，compaction点的管理等等。

### 11.1 VersionSet接口

**1 首先是构造函数，VersionSet会使用到TableCache，这个是调用者传入的。TableCache用于Get k/v操作。**

```
VersionSet(const std::string& dbname, const Options* options,
           TableCache*table_cache, const InternalKeyComparator*);
```

`VersionSet`的构造函数很简单，除了根据参数初始化，还有两个地方值得注意：
**N1** next_file_number_从2开始；
**N2** 创建新的Version并加入到Version链表中，并设置CURRENT=新创建version；
其它的数字初始化为0，指针初始化为NULL。
**2 恢复函数，从磁盘恢复最后保存的元信息**

```
Status Recover();
```

**3 标记指定的文件编号已经被使用了**

```
void MarkFileNumberUsed(uint64_t number);
```

逻辑很简单，就是根据编号更新文件编号计数器：

```
if (next_file_number_ <= number) 
     next_file_number_ = number + 1;
```

**4 在current version上应用指定的VersionEdit，生成新的MANIFEST信息，保存到磁盘上，并用作current version。**
要求：没有其它线程并发调用；要用于mu；

```
Status LogAndApply(VersionEdit* edit, port::Mutex* mu)EXCLUSIVE_LOCKS_REQUIRED(mu);
```

**5 对于@v中的@key，返回db中的大概位置**

```
uint64_t ApproximateOffsetOf(Version* v, const InternalKey& key);
```

**6 其它一些简单接口，信息获取或者设置，如下：**

```
//返回current version
Version* current() const {
     return current_; 
}   

// 当前的MANIFEST文件号  
uint64_t ManifestFileNumber() const {
     return manifest_file_number_;
} 

// 分配并返回新的文件编号  
uint64_t NewFileNumber() {
     return next_file_number_++; 
} 

// 返回当前log文件编号  
uint64_t LogNumber() const { 
     return log_number_; 
}

// 返回正在compact的log文件编号，如果没有返回0  
uint64_t PrevLogNumber() const {
     return prev_log_number_; 
}

// 获取、设置last sequence，set时不能后退  
uint64_t LastSequence() const {
     return last_sequence_; 
}

void SetLastSequence(uint64_t s) {  
    assert(s >=last_sequence_);  
    last_sequence_ = s;  
}  

// 返回指定level中所有sstable文件大小的和  
int64_t NumLevelBytes(int level) const;  

// 返回指定level的文件个数  
int NumLevelFiles(int level) const;  

// 重用@file_number，限制很严格：@file_number必须是最后分配的那个  
// 要求: @file_number是NewFileNumber()返回的.  

void ReuseFileNumber(uint64_t file_number) {  
    if (next_file_number_ ==file_number + 1) next_file_number_ = file_number;  
}  

// 对于所有level>0，遍历文件，找到和下一层文件的重叠数据的最大值(in bytes)  
// 这个就是Version:: GetOverlappingInputs()函数的简单应用  
int64_t MaxNextLevelOverlappingBytes(); 

// 获取函数，把所有version的所有level的文件加入到@live中  
void AddLiveFiles(std::set<uint64_t>* live);  

// 返回一个可读的单行信息——每个level的文件数，保存在*scratch中  
struct LevelSummaryStorage {char buffer[100]; };  
const char* LevelSummary(LevelSummaryStorage* scratch) const; 
```


下面就来分析这两个接口`Recover`、`LogAndApply`以及`ApproximateOffsetOf`。

### 11.2 VersionSet::Builder类

`Builder`是一个**内部辅助类**，其主要作用是：
1 把一个`MANIFEST`记录的元信息应用到版本管理器`VersionSet`中；
2 把当前的版本状态设置到一个Version对象中。

**11.2.1 成员与构造**

`Builder`的vset_与base_都是调用者传入的，此外它还为`FileMetaData`定义了一个比较类`BySmallestKey`，首先依照文件的min key，小的在前；如果min key相等则`file number`小的在前。

```
typedefstd::set<FileMetaData*, BySmallestKey> FileSet;  
// 这个是记录添加和删除的文件  
struct LevelState { 
  std::set<uint64_t>deleted_files;  
  // 保证添加文件的顺序是有效定义的
  FileSet* added_files;   
};  
VersionSet* vset_;  
Version* base_;  
LevelStatelevels_[config::kNumLevels];  

// 其接口有3个：  
void Apply(VersionEdit* edit);  
void SaveTo(Version* v);  
void MaybeAddFile(Version* v, int level, FileMetaData* f);  
```

构造函数执行简单的初始化操作，在析构时，遍历检查`LevelState::added_files`，如果文件引用计数为0，则删除文件。

**11.2.2 Apply()**

函数声明：`voidApply(VersionEdit* edit)`，该函数将edit中的修改应用到当前状态中。注意除了`compaction`点直接修改了vset_，其它删除和新加文件的变动只是先存储在Builder自己的成员变量中，在调用SaveTo(v)函数时才施加到v上。

**S1 把edit记录的compaction点应用到当前状态**

```
edit->compact_pointers_ => vset_->compact_pointer_
```

**S2 把edit记录的已删除文件应用到当前状态**

```
edit->deleted_files_ => levels_[level].deleted_files
```

**S3把edit记录的新加文件应用到当前状态，这里会初始化文件的allowed_seeks值，以在文件被无谓seek指定次数后自动执行compaction，这里作者阐述了其设置规则。**

```
for (size_t i = 0; i <edit->new_files_.size(); i++) {  
    const int level =edit->new_files_[i].first;  
    FileMetaData* f = newFileMetaData(edit->new_files_[i].second);  
    f->refs = 1;  
    f->allowed_seeks = (f->file_size /16384); // 16KB-见下面  
    if (f->allowed_seeks <100) f->allowed_seeks = 100;  
    levels_[level].deleted_files.erase(f->number); // 以防万一  
    levels_[level].added_files->insert(f);  
}
```


值allowed_seeks事关compaction的优化，其计算依据如下，首先假设：

> 1 一次seek时间为10ms
> 2 写入10MB数据的时间为10ms（100MB/s）
> 3 compact 1MB的数据需要执行25MB的IO
> ->从本层读取1MB
> ->从下一层读取10-12MB（文件的key range边界可能是非对齐的）
> ->向下一层写入10-12MB

这意味这25次seek的代价等同于`compact 1MB`的数据，也就是一次seek花费的时间大约相当于`compact 40KB`的数据。基于保守的角度考虑，对于每16KB的数据，我们允许它在触发compaction之前能做一次seek。

**11.2.3 MaybeAddFile()**

函数声明：

```
voidMaybeAddFile(Version* v, int level, FileMetaData* f);
```

该函数尝试将f加入到`levels_[level]`文件set中。
要满足两个条件：

> 1 文件不能被删除，也就是不能在levels_[level].deleted_files集合中；
> 2 保证文件之间的key是连续的，即基于比较器vset_->icmp_，f的min key要大于levels_[level]集合中最后一个文件的max key；

**11.2.4 SaveTo()**

把当前的状态存储到v中返回，函数声明：

```
void SaveTo(Version* v);
```

函数逻辑：For循环遍历所有的`level[0, config::kNumLevels-1]`，把新加的文件和已存在的文件merge在一起，丢弃已删除的文件，结果保存在v中。对于level> 0，还要确保集合中的文件没有重合。
**S1 merge流程**

```
// 原文件集合
conststd::vector<FileMetaData*>& base_files = base_->files_[level];   
std::vector<FileMetaData*>::const_iterator base_iter =base_files.begin();  
std::vector<FileMetaData*>::const_iterator base_end =base_files.end();  
const FileSet* added =levels_[level].added_files;  
v->files_[level].reserve(base_files.size()+ added->size());  
for (FileSet::const_iteratoradded_iter = added->begin();
    added_iter !=added->end(); ++added_iter) {  
    //加入base_中小于added_iter的那些文件  

      for(std::vector<FileMetaData*>::const_iterator bpos = std::upper_bound(base_iter,base_end,*added_iter, cmp);  
        base_iter != bpos;++base_iter) { 
        // base_iter逐次向后移到  
        MaybeAddFile(v, level,*base_iter);  
    }  
    // 加入added_iter  
    MaybeAddFile(v, level,*added_iter); 
 }  

   // 添加base_剩余的那些文件  
   for (; base_iter != base_end;++base_iter) 
      MaybeAddFile(v, level, *base_iter);
```


对象cmp就是前面定义的比较仿函数`BySmallestKey`对象。
**S2 检查流程，保证level>0的文件集合无重叠，基于vset_->icmp_，确保文件i-1的max key < 文件i的min key。**

### 11.3 Recover()

对于`VersionSet`而言，`Recover`就是根据`CURRENT`指定的`MANIFEST`，读取db元信息。这是9.3介绍的Recovery流程的开始部分。

**11.3.1 函数流程**

下面就来分析其具体逻辑。

**S1 读取CURRENT文件，获得最新的MANIFEST文件名，根据文件名打开MANIFEST文件。CURRENT文件以\n结尾，读取后需要trim下。**

```
std::string current; // MANIFEST文件名  
ReadFileToString(env_, CurrentFileName(dbname_), ¤t);  
std::string dscname = dbname_ + "/" + current;  
SequentialFile* file;  
env_->NewSequentialFile(dscname, &file); 
```


**S2 读取MANIFEST内容，MANIFEST是以log的方式写入的，因此这里调用的是log::Reader来读取。然后调用VersionEdit::DecodeFrom，从内容解析出VersionEdit对象，并将VersionEdit记录的改动应用到versionset中。读取MANIFEST中的log number, prev log number, nextfile number, last sequence。**

```
Builder builder(this, current_);  
while (reader.ReadRecord(&record, &scratch) && s.ok()) {  
      VersionEdit edit;  
      s = edit.DecodeFrom(record);  
      if (s.ok())builder.Apply(&edit);  
      // log number, file number, …逐个判断  
      if (edit.has_log_number_) { 
          log_number =edit.log_number_;  
          have_log_number = true;  
      }  
      … …  
}  
```

**S3 将读取到的log number, prev log number标记为已使用。**

```
MarkFileNumberUsed(prev_log_number);
MarkFileNumberUsed(log_number);
```

**S4 最后，如果一切顺利就创建新的Version，并应用读取的几个number。**

```
if (s.ok()) {  
    Version* v = newVersion(this);  
    builder.SaveTo(v);  
    // 安装恢复的version  
    Finalize(v);  
    AppendVersion(v);  
    manifest_file_number_ =next_file;  
    next_file_number_ = next_file+ 1;  
    last_sequence_ = last_sequence;  
    log_number_ = log_number;  
    prev_log_number_ =prev_log_number;  
}  
```

`Finalize(v)`和`AppendVersion(v)`用来安装并使用version v，在`AppendVersion`函数中会将`current version`设置为v。下面就来分别分析这两个函数。

**11.3.2 Finalize()**

函数声明：

```
void Finalize(Version*v);
```

该函数依照规则为下次的compaction计算出最适用的level，对于level 0和>0需要分别对待，逻辑如下。

**S1 对于level 0以文件个数计算，kL0_CompactionTrigger默认配置为4。**

```
score =v->files_[level].size()/static_cast<double>(config::kL0_CompactionTrigger);
```

**S2 对于level>0，根据level内的文件总大小计算**

```
const uint64_t level_bytes = TotalFileSize(v->files_[level]);
score = static_cast<double>(level_bytes) /MaxBytesForLevel(level);
```

**S3 最后把计算结果保存到v的两个成员compaction_level_和compaction_score_中。**

其中函数MaxBytesForLevel根据level返回其本层文件总大小的预定最大值。
计算规则为：**1048576.0\* level^10**。
这里就有一个问题，为何level0和其它level计算方法不同，原因如下，这也是leveldb为`compaction`所做的另一个优化。

> 1 对于较大的写缓存（write-buffer），做太多的level 0 compaction并不好
> 2 每次read操作都要merge level 0的所有文件，因此我们不希望level 0有太多的小文件存在（比如写缓存太小，或者压缩比较高，或者覆盖/删除较多导致小文件太多）。
> 看起来这里的写缓存应该就是配置的操作log大小。

**11.3.3 AppendVersion()**

函数声明：

```
void AppendVersion(Version*v);
```

把v加入到`versionset`中，并设置为`current version`。并对老的`current version`执行Uref()。
在双向循环链表中的位置在`dummy_versions_`之前。
