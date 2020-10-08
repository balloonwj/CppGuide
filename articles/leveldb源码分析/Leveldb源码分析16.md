# Leveldb源码分析16

本系列《leveldb源码分析》共有22篇文章，这是第十六篇

## 10.Version分析之一

先来**分析leveldb对单版本的sstable文件管理**，主要集中在Version类中。前面的10.4节已经说明了Version类的功能和成员，这里分析其函数接口和代码实现。
**Version不会修改其管理的sstable文件，只有读取操作。**

### 10.1 Version接口

先来看看Version类的接口函数，接下来再一一分析。

```
// 追加一系列iterator到 @*iters中，
//将在merge到一起时生成该Version的内容  

// 要求: Version已经保存了(见VersionSet::SaveTo)  

void AddIterators(constReadOptions&, 
                  std::vector<Iterator*>* iters);  


// 给定@key查找value，如果找到保存在@*val并返回OK。  

// 否则返回non-OK，设置@ *stats.  

// 要求：没有hold lock  

struct GetStats {  

  FileMetaData* seek_file;  

  int seek_file_level;  

};  

Status Get(constReadOptions&, const LookupKey& key, 
           std::string* val,GetStats* stats);  



// 把@stats加入到当前状态中，如果需要触发新的compaction返回true  

// 要求：hold lock  

bool UpdateStats(constGetStats& stats);  

void GetOverlappingInputs(intlevel,  

    const InternalKey*begin,         // NULL 指在所有key之前  

    const InternalKey* end,           // NULL指在所有key之后  

    std::vector<FileMetaData*>* inputs);  



// 如果指定level中的某些文件和[*smallest_user_key,*largest_user_key]
//有重合就返回true。  

// @smallest_user_key==NULL表示比DB中所有key都小的key.  

// @largest_user_key==NULL表示比DB中所有key都大的key.  

bool OverlapInLevel(int level,const Slice*smallest_user_key,
                    const Slice* largest_user_key);  



// 返回我们应该在哪个level上放置新的memtable compaction，  

// 该compaction覆盖了范围[smallest_user_key,largest_user_key].  

int PickLevelForMemTableOutput(const Slice& smallest_user_key,  

                           const Slice& largest_user_key);  


 // 指定level的sstable个数 
int NumFiles(int level) const {return files_[level].size();
```

### 10.2 Version::AddIterators()

该函数最终在DB::NewIterators()接口中被调用，调用层次为：
`DBImpl::NewIterator()->DBImpl::NewInternalIterator()->Version::AddIterators()`。
函数功能是为该Version中的所有sstable都创建一个Two Level Iterator，以遍历sstable的内容。

- 对于**level=0**级别的sstable文件，直接通过TableCache::NewIterator()接口创建，这会直接载入sstable文件到内存cache中。
- 对于**level>0**级别的sstable文件，通过函数NewTwoLevelIterator()创建一个TwoLevelIterator，这就使用了lazy open的机制。

下面来分析函数代码：

#### S1 对于level=0级别的sstable文件，直接装入cache，level0的sstable文件可能有重合，需要merge。

```
for (size_t i = 0; i <files_[0].size(); i++) {  
	iters->push_back(vset_->table_cache_->NewIterator(// versionset::table_cache_  
		options,files_[0][i]->number, files_[0][i]->file_size));  
}  
```

#### S2 对于level>0级别的sstable文件，lazy open机制，它们不会有重叠。

```
for (int ll = 1; ll <config::kNumLevels; ll++) {  
	if(!files_[ll].empty()) iters->push_back(NewConcatenatingIterator(options,level));  
}  
```

函数NewConcatenatingIterator()直接返回一个TwoLevelIterator对象：

```
return NewTwoLevelIterator(new LevelFileNumIterator(vset_->icmp_,&files_[level]),
                           &GetFileIterator,vset_->table_cache_, options);
```

- 其第一级iterator是一个LevelFileNumIterator
- 第二级的迭代函数是GetFileIterator

下面就来分别分析之。
GetFileIterator是一个静态函数，很简单，直接返回TableCache::NewIterator()。函数声明为：

```
static Iterator* GetFileIterator(void* arg,const ReadOptions& options, constSlice& file_value)
TableCache* cache =reinterpret_cast<TableCache*>(arg);  

	if (file_value.size() != 16) { // 错误  
		return NewErrorIterator(Status::Corruption("xxx"));  

} else {  

	return cache->NewIterator(options,  

                DecodeFixed64(file_value.data()), // filenumber  

                DecodeFixed64(file_value.data() + 8)); // filesize  

}  
```

这里的**file_value**是取自于LevelFileNumIterator的value，它的value()函数把file number和size以Fixed 8byte的方式压缩成一个Slice对象并返回。

### 10.3 Version::LevelFileNumIterator类

这也是一个继承者Iterator的子类，一个内部Iterator。

**给定一个version/level对**，生成该level内的文件信息。

**对于给定的entry**：

- key()返回的是文件中所包含的最大的key；
- value()返回的是|file number(8 bytes)|file size(8 bytes)|串；
- 它的构造函数接受两个参数：InternalKeyComparator&，用于key的比较；
- vector<FileMetaData*>*，指向version的所有sstable文件列表。

```
LevelFileNumIterator(const InternalKeyComparator& icmp,
                                      const std::vector<FileMetaData*>* flist)
                                      : icmp_(icmp), flist_(flist),index_(flist->size()) {} // Marks as invalid
```

来看看其接口实现，不限啰嗦，全部都列出来。

Valid函数、SeekToxx和Next/Prev函数都很简单，毕竟容器是一个vector。Seek函数调用了FindFile，这个函数后面会分析。

```
virtual void Seek(constSlice& target) { index_ = FindFile(icmp_, *flist_, target);}  

virtual void SeekToFirst() {index_ = 0; }  

virtual void SeekToLast() {index_ = flist_->empty() ? 0 : flist_->size() - 1;}  

virtual void Next() {  

    assert(Valid());  

    index_++;  

}  


virtual void Prev() {  

    assert(Valid());  

    if (index_ == 0) index_ =flist_->size(); // Marks as invalid  

    else index_--;  

}  


Slice key() const {  

	assert(Valid());  

	return(*flist_)[index_]->largest.Encode(); // 返回当前sstable包含的largest key  

}  


Slice value() const { // 根据|number|size|的格式Fixed int压缩  

    assert(Valid());  

    EncodeFixed64(value_buf_,(*flist_)[index_]->number);  

    EncodeFixed64(value_buf_+8,(*flist_)[index_]->file_size);  

    return Slice(value_buf_,sizeof(value_buf_));  

}  
```

来看FindFile，这其实是一个二分查找函数，因为传入的sstable文件列表是有序的，因此可以使用二分查找算法。就不再列出代码了。

### 10.4 Version::Get()

查找函数，直接在DBImpl::Get()中被调用，函数原型为：

```
Status Version::Get(const ReadOptions& options, constLookupKey& k, std::string* value, GetStats* stats)
```

**如果本次Get不止seek了一个文件**（仅会发生在level 0的情况），就将搜索的第一个文件保存在stats中。**如果stat有数据返回**，表明本次读取在搜索到包含key的sstable文件之前，还做了其它无谓的搜索。这个结果将用在UpdateStats()中。
这个函数逻辑还是有些复杂的，来看看代码。

#### S1 首先，取得必要的信息，初始化几个临时变量

```
Slice ikey = k.internal_key();  

Slice user_key = k.user_key();  

const Comparator* ucmp =vset_->icmp_.user_comparator();  

Status s;  

stats->seek_file = NULL;  

stats->seek_file_level = -1;  

FileMetaData* last_file_read =NULL; // 在找到>1个文件时，读取时记录上一个  

int last_file_read_level = -1;        // 这仅发生在level 0的情况下  

std::vector<FileMetaData*>tmp;  

FileMetaData* tmp2;  
```

#### S2 从0开始遍历所有的level，依次查找。因为entry不会跨越level，因此如果在某个level中找到了entry，那么就无需在后面的level中查找了。

```
for (int level = 0; level <config::kNumLevels; level++) {  

    size_t num_files = files_[level].size();  

    if (num_files == 0) continue; // 本层没有文件，则直接跳过  

    // 取得level下的所有sstable文件列表，搜索本层  

    FileMetaData* const* files = &files_[level][0];   
```

后面的所有逻辑都在for循环体中。

#### S3 遍历level下的sstable文件列表，搜索，注意对于level=0和>0的sstable文件的处理，由于level 0文件之间的key可能有重叠，因此处理逻辑有别于>0的level。

##### S3.1 对于level 0，文件可能有重叠，找到所有和user_key有重叠的文件，然后根据时间顺序从最新的文件依次处理。

```
tmp.reserve(num_files);  

for (uint32_t i = 0; i <num_files; i++) { // 遍历level 0下的所有sstable文件  

    FileMetaData* f =files[i];  

    if(ucmp->Compare(user_key, f->smallest.user_key()) >= 0 &&  

    ucmp->Compare(user_key, f->largest.user_key()) <= 0)  

    tmp.push_back(f); // sstable文件有user_key有重叠   

}  

if (tmp.empty()) continue;  

std::sort(tmp.begin(),tmp.end(), NewestFirst); // 排序  

files = &tmp[0]; num_files= tmp.size();// 指向tmp指针和大小  
```

##### S3.2 对于level>0，leveldb保证sstable文件之间不会有重叠，所以处理逻辑有别于level 0，直接根据ikey定位到sstable文件即可。

```
//二分查找，找到第一个largest key >=ikey的file index  

uint32_t index =FindFile(vset_->icmp_, files_[level], ikey);  

if (index >= num_files) { // 未找到，文件不存在  

  files = NULL;  num_files = 0;  

} else {  

    tmp2 = files[index];  

    if(ucmp->Compare(user_key, tmp2->smallest.user_key()) < 0) {  

        // 找到的文件其所有key都大于user_key，等于文件不存在  

        files = NULL;  num_files = 0;  

	} else {  

		files = &tmp2;  num_files = 1;  

	}  

}  
```

#### S4 遍历找到的文件，存在files中，其个数为num_files。

```
for (uint32_t i = 0; i <num_files; ++i) {
```

后面的逻辑都在这一层循环中，只要在某个文件中找到了k/v对，就跳出for循环。

##### S4.1 如果本次读取不止搜索了一个文件，记录之，这仅会发生在level 0的情况下。

```
if(last_file_read != NULL && stats->seek_file == NULL) {  

   // 本次读取不止seek了一个文件，记录第一个  

   stats->seek_file =last_file_read;  

   stats->seek_file_level= last_file_read_level;  

}  

FileMetaData* f = files[i];  

last_file_read = f; // 记录本次读取的level和file  

last_file_read_level =level;  
```

##### S4.2 调用TableCache::Get()尝试获取{ikey, value}，如果返回OK则进入，否则直接返回，传递的回调函数是SaveValue()。

```
Saver saver; // 初始化saver  

saver.state = kNotFound;  

saver.ucmp = ucmp;  

saver.user_key = user_key;  

saver.value = value;  

s = vset_->table_cache_->Get(options,f->number, f->file_size,  

                            ikey, &saver, SaveValue);  

if (!s.ok()) return s;  
```

##### S4.3 根据saver的状态判断，如果是Not Found则向下搜索下一个更早的sstable文件，其它值则返回。

```
switch (saver.state) {  

  case kNotFound: break; // 继续搜索下一个更早的sstable文件  

  case kFound:  return s; // 找到  

  case kDeleted: // 已删除  

        s =Status::NotFound(Slice());  // 为了效率，使用空的错误字符串  

        return s;  

  case kCorrupt: // 数据损坏  

        s =Status::Corruption("corrupted key for ", user_key);  

        return s;  
}  
```

以上就是Version::Get()的代码逻辑，如果level 0的sstable文件太多的话，会影响读取速度，这也是为什么进行compaction的原因。
另外，还有一个传递给TableCache::Get()的saver函数，下面就来简单分析下。这是一个静态函数：static void SaveValue(void* arg,const Slice& ikey, const Slice& v)。它内部使用了结构体Saver：

```
struct Saver {
    SaverState state;
    const Comparator* ucmp; // user key比较器
    Slice user_key;
    std::string* value;
};
```

函数SaveValue的逻辑很简单。**首先**解析Table传入的InternalKey，**然后**根据指定的Comparator判断user key是否是要查找的user key。**如果**是并且type是kTypeValue，则**设置**到Saver::*value中，并**返回**kFound，否则返回kDeleted。代码如下：

```
Saver* s =reinterpret_cast<Saver*>(arg);  

ParsedInternalKey parsed_key; // 解析ikey到ParsedInternalKey  

if (!ParseInternalKey(ikey,&parsed_key)) s->state = kCorrupt; // 解析失败  

else {  

    if(s->ucmp->Compare(parsed_key.user_key, s->user_key) == 0) { // 比较user key  

        s->state =(parsed_key.type == kTypeValue) ? kFound : kDeleted;  

        if (s->state == kFound) s->value->assign(v.data(), v.size()); // 找到，保存结果  

    }  

}  
```

下面要分析的几个函数，或多或少都和compaction相关。
