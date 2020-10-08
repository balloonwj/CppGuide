# leveldb源码分析11

本系列《leveldb源码分析》共有22篇文章，这是第十一篇

## 7.TableCache 

这章的内容比较简单，篇幅也不长。

### 7.1 TableCache简介

**TableCache缓存**的是Table对象，每个DB一个，它内部使用一个LRUCache缓存所有的table对象，实际上其内容是文件编号{file number, TableAndFile}*。*TableAndFile是一个拥有**2个变量**的结构体：RandomAccessFile和Table*；

TableCache类的**主要成员变量**有：

```
Env* const env_;            // 用来操作文件  
const std::string dbname_;  // db名  
Cache* cache_;              // LRUCache
```


三个函数接口，其中的参数**@file_number**是文件编号，**@file_size**是文件大小：

```
void Evict(uint64_tfile_number);  
// 该函数用以清除指定文件所有cache的entry，
//函数实现很简单，就是根据file number清除cache对象。  
EncodeFixed64(buf,file_number); cache_->Erase(Slice(buf, sizeof(buf)));  
Iterator* NewIterator(constReadOptions& options, uint64_t file_number,  
                      uint64_t file_size, Table**tableptr = NULL);  
//该函数为指定的file返回一个iterator(对应的文件长度必须是"file_size"字节). 
//如果tableptr不是NULL，那么tableptr保存的是底层的Table指针。
//返回的tableptr是cache拥有的，不能被删除，生命周期同返回的iterator

Status Get(constReadOptions& options,  
           uint64_t file_number,uint64_t file_size,  
           const Slice& k,void* arg,  
           void(*handle_result)(void*, const Slice&, const Slice&)); 

// 这是一个查找函数，如果在指定文件中seek 到internal key "k" 找到一个entry，
//就调用 (*handle_result)(arg,found_key, found_value).
```

### 7.2 TableCache::Get()

先来看看**Get接口，**只有几行代码：

```
Cache::Handle* handle = NULL;  
Status s =FindTable(file_number, file_size, &handle);  
if (s.ok()) 
{  
    Table* t =reinterpret_cast<TableAndFile*>(cache_->Value(handle))->table;  
    s = t->InternalGet(options,k, arg, saver);  
    cache_->Release(handle);  
}  
return s;
```


**首先**根据file_number找到Table的cache对象，**如果**找到了就调用Table::InternalGet，对查找结果的处理在调用者传入的**saver回调函数**中。
Cache在Lookup找到**cache对象**后，**如果**不再使用需要调用Release减引用计数。这个见Cache的接口说明。

### 7.3 TableCache遍历

函数NewIterator()，返回一个可以遍历**Table对象的Iterator指针**，函数逻辑：

#### S1 初始化tableptr，调用FindTable，返回cache对象

```
if (tableptr != NULL) *tableptr =NULL;  
Cache::Handle* handle = NULL;  
Status s =FindTable(file_number, file_size, &handle);  
if (!s.ok()) returnNewErrorIterator(s);
```

#### S2 从cache对象中取出Table对象指针，调用其NewIterator返回Iterator对象，并为Iterator注册一个cleanup函数。

```
Table* table =reinterpret_cast<TableAndFile*>(cache_->Value(handle))->table;  
Iterator* result =table->NewIterator(options);  
result->RegisterCleanup(&UnrefEntry, cache_, handle);  
if (tableptr != NULL) *tableptr= table;  
return result;
```

### 7.4 TableCache::FindTable()

前面的**遍历和Get函数**都依赖于FindTable这个私有函数完成对cache的查找，下面就来看看该函数的逻辑。函数声明为：

```
Status FindTable(uint64_t file_number, uint64_t file_size,
                 Cache::Handle** handle)
```

函数流程为：

#### S1 首先根据file number从cache中查找table，找到就直接返回成功。

```
char buf[sizeof(file_number)];  
EncodeFixed64(buf, file_number);  
Slice key(buf, sizeof(buf));  
*handle = cache_->Lookup(key);
```

#### S2 如果没有找到，说明table不在cache中，则根据file number和db name打开一个RadomAccessFile。Table文件格式为：..sst。如果文件打开成功，则调用Table::Open读取sstable文件。

```
std::string fname =TableFileName(dbname_, file_number);  
RandomAccessFile* file = NULL;  
Table* table = NULL;  
s =env_->NewRandomAccessFile(fname, &file);  
if (s.ok()) s =Table::Open(*options_, file, file_size, &table);
```

#### S3 如果Table::Open成功则，插入到Cache中。

```
TableAndFile* tf = newTableAndFile(table, file);  
*handle = cache_->Insert(key,tf, 1, &DeleteEntry); 
```


如果失败，则**删除file**，直接返回失败，失败的结果是不会cache的。

### 7.5 辅助函数

有点啰嗦，不过还是写一下吧。其中一个是为**LRUCache注册的删除函数DeleteEntry。**

```
static void DeleteEntry(const Slice& key, void* value) 
{  
    TableAndFile* tf =reinterpret_cast<TableAndFile*>(value);  
    delete tf->table;  
    delete tf->file;  
    delete tf;  
} 
```


另外一个是**为Iterator注册的清除函数UnrefEntry。**

```
static void UnrefEntry(void* arg1, void* arg2) 
{  
    Cache* cache =reinterpret_cast<Cache*>(arg1);  
    Cache::Handle* h =reinterpret_cast<Cache::Handle*>(arg2);  
    cache->Release(h);  
}
```
