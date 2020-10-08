# leveldb源码分析17

本系列《leveldb源码分析》共有22篇文章，这是第十七篇

**10 Version分析之2**

### 10.5 Version::UpdateStats()

当`Get`操作直接搜寻`memtable`没有命中时，就需要调用`Version::Get()`函数从磁盘load数据文件并查找。如果此次Get不止seek了一个文件，就记录第一个文件到stat并返回。其后leveldb就会调用`UpdateStats(stat)`。
`Stat`表明在指定key range查找key时，都要先**seek此文件**，才能在后续的sstable文件中找到`key`。
该函数是将stat记录的sstable文件的`allowed_seeks`减1，减到0就执行compaction。也就是说如果文件被seek的次数超过了限制，表明读取效率已经很低，需要执行compaction了。所以说`allowed_seeks`是对compaction流程的有一个优化。
函数声明：`boolVersion::UpdateStats(const GetStats& stats)`
函数逻辑很简单：

```
FileMetaData* f =stats.seek_file;  
if (f != NULL) {  
   f->allowed_seeks--;  
   if (f->allowed_seeks <=0 && file_to_compact_ == NULL) {  
       file_to_compact_ = f;  
       file_to_compact_level_ =stats.seek_file_level;  
       return true;  
  }  
}  
return false;
```


变量`allowed_seeks`的值在sstable文件加入到`version`时确定，也就是后面将遇到的`VersionSet::Builder::Apply()`函数。

### 10.6 Version::GetOverlappingInputs()

它在指定level中找出和**[begin, end]**有重合的sstable文件，函数声明为：

```
void Version::GetOverlappingInputs(int level,
           const InternalKey* begin, constInternalKey* end, std::vector<FileMetaData*>* inputs);
```

要注意的是，对于`level0`，由于文件可能有重合，其处理具有特殊性。当在level 0中找到有sstable文件和**[begin, end]**重合时，会相应的将`begin/end`扩展到文件的min key/max key，然后重新开始搜索。
了解了功能，下面分析函数实现代码，逻辑还是很直观的。
S1 首先根据参数初始化查找变量。

```
inputs->clear();  
Slice user_begin, user_end;  
if (begin != NULL) user_begin =begin->user_key();  
if (end != NULL)  user_end = end->user_key();  
const Comparator* user_cmp =vset_->icmp_.user_comparator(); 
```


S2 遍历该层的sstable文件，比较sstable的**{minkey,max key}**和传入的**[begin, end]**，如果有重合就记录文件到`@inputs`中，需要对level 0做特殊处理。

```
for (size_t i = 0; i <files_[level].size(); ) {  
    FileMetaData* f =files_[level][i++];  
    const Slice file_start =f->smallest.user_key();  
    const Slice file_limit =f->largest.user_key();  
    if (begin != NULL &&user_cmp->Compare(file_limit, user_begin) < 0) {  
       //"f" 中的k/v全部在指定范围之前; 跳过  
    } else if (end != NULL&& user_cmp->Compare(file_start, user_end) > 0) {  
       //"f" 中的k/v全部在指定范围之后; 跳过  
    } else {  
       inputs->push_back(f); // 有重合，记录  
       if (level == 0) {  
         // 对于level 0，sstable文件可能相互有重叠，所以要检查新加的文件  
         // 是否范围更大，如果是则扩展范围重新开始搜索  
         if (begin != NULL&& user_cmp->Compare(file_start, user_begin) < 0) {  
             user_begin = file_start;  
             inputs->clear();  
             i = 0;  
         } else if (end != NULL&& user_cmp->Compare(file_limit, user_end) > 0) {  
             user_end = file_limit;  
             inputs->clear();  
             i = 0;  
            }  
         }  
     }  
}  
```

### 10.7 Version::OverlapInLevel()

检查是否和指定level的文件有重合，该函数直接调用了`SomeFileOverlapsRange()`，这两个函数的声明为：

```
bool Version::OverlapInLevel(int level,const Slice*smallest_user_key, 
                             const Slice* largest_user_key){
      return SomeFileOverlapsRange(vset_->icmp_,(level > 0), files_[level],
                                   smallest_user_key, largest_user_key);
}

bool SomeFileOverlapsRange(const InternalKeyComparator& icmp, 
							 bool disjoint_sorted_files,
                             const std::vector<FileMetaData*>& files,const 	
                             Slice*smallest_user_key, 
                             const Slice* largest_user_key);
```

所以下面直接分析`SomeFileOverlapsRange()`函数的逻辑，代码很直观。
`disjoint_sorted_files=true`，表明文件集合是互不相交、有序的，对于乱序的、可能有交集的文件集合，需要逐个查找，找到有重合的就返回true；对于有序、互不相交的文件集合，直接执行二分查找。

```
// S1 乱序、可能相交的文件集合，依次查找  
for (size_t i = 0; i <files.size(); i++) {  
     const FileMetaData* f =files[i];  
     if(AfterFile(ucmp,smallest_user_key, f) ||
        BeforeFile(ucmp, largest_user_key, f)){  

      } else 
         return true;  // 有重合  
}  

// S2 有序&互不相交，直接二分查找  
uint32_t index = 0;  
if (smallest_user_key != NULL) {  
     // Findthe earliest possible internal key smallest_user_key  
     InternalKeysmall(*smallest_user_key, kMaxSequenceNumber,kValueTypeForSeek);  
     index = FindFile(icmp, files,small.Encode());  
 }  
 if (index >= files.size())
     // 不存在比smallest_user_key小的key
     return false;   
 //保证在largest_user_key之后
 return !BeforeFile(ucmp,largest_user_key, files[index]); 
```


上面的逻辑使用到了`AfterFile()`和`BeforeFile()`两个辅助函数，都很简单。

```
static bool AfterFile(const Comparator* ucmp,  
                       const Slice* user_key, constFileMetaData* f) {  
     return (user_key!=NULL&& ucmp->Compare(*user_key, f->largest.user_key())>0);  
}  

static bool BeforeFile(const Comparator* ucmp,  
constSlice* user_key, const FileMetaData* f) {  
	return (user_key!=NULL&& ucmp->Compare(*user_key, f->smallest.user_key())<0);  
}  
```

### 10.8 Version::PickLevelForMemTableOutput()

函数返回我们应该在哪个level上放置新的`memtable compaction`，这个compaction覆盖了范围**[smallest_user_key,largest_user_key]**。
该函数的调用链为：

```
DBImpl::RecoverLogFile/DBImpl::CompactMemTable -> DBImpl:: WriteLevel0Table->Version::PickLevelForMemTableOutput;
```

函数声明如下：

```
int Version::PickLevelForMemTableOutput(const Slice& smallest_user_key, constSlice& largest_user_key);
```

如果`level 0`没有找到重合就向下一层找，最大查找层次为`kMaxMemCompactLevel = 2`。如果在level 0or1找到了重合，就返回level 0。否则查找level 2，如果level 2有重合就返回level 1，否则返回level 2。
函数实现：

```
int level = 0;  
//level 0无重合  
if (!OverlapInLevel(0,&smallest_user_key, &largest_user_key)) { 
    // 如果下一层没有重叠，就压到下一层，  
    // andthe #bytes overlapping in the level after that are limited.  
    InternalKeystart(smallest_user_key, kMaxSequenceNumber, kValueTypeForSeek);  
    InternalKeylimit(largest_user_key, 0, static_cast<ValueType>(0));  
    std::vector<FileMetaData*> overlaps;  
    while (level <config::kMaxMemCompactLevel) {  
       if (OverlapInLevel(level +1, &smallest_user_key, &largest_user_key))  
           break; // 检查level + 1层，有重叠就跳出循环  
       GetOverlappingInputs(level +2, &start, &limit, &overlaps); // 没理解这个调用  
       const int64_t sum =TotalFileSize(overlaps);  
       if (sum >kMaxGrandParentOverlapBytes) break;  
       level++;  
    }  
}  
return level;
```


这个函数在整个`compaction`逻辑中的作用在分析DBImpl时再来结合整个流程分析，现在只需要了解它找到一个level存放新的compaction就行了。
如果返回**level = 0**，表明在level 0或者1和指定的range有重叠；如果返回1，表明在level2和指定的range有重叠；否则就返回2（`kMaxMemCompactLevel`）。
也就是说在`compactmemtable`的时候，写入的sstable文件不一定总是在level 0，如果比较顺利，没有重合的，它可能会写到level1或者level2中。

### 10.9 小结

`Version`是管理某个版本的所有`sstable`的类，就其导出接口而言，无非是遍历sstable，查找k/v。以及为`compaction`做些事情，给定range，检查重叠情况。
而它不会修改它管理的sstable这些文件，对这些文件而言它是只读操作接口。
