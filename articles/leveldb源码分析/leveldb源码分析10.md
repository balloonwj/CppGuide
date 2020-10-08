# leveldb源码分析10

本系列《leveldb源码分析》共有22篇文章，这是第十篇

## 6.SSTable之四 

### 6.6 遍历Table

#### 6.6.1 遍历接口

Table导出了一个返回**Iterator**的接口，通过Iterator对象，调用者就可以**遍历**Table的内容，它简单的返回了一个**TwoLevelIterator**对象。见函数实现：

```
Iterator* NewIterator(const ReadOptions&options) const;  
{  
     return NewTwoLevelIterator(rep_->index_block->NewIterator(rep_->options.comparator),  
                                &Table::BlockReader,const_cast<Table*>(this), options);  
}  
// 函数NewTwoLevelIterator创建了一个TwoLevelIterator对象：  
Iterator* NewTwoLevelIterator(Iterator* index_iter,BlockFunction block_function,  
                              void* arg, constReadOptions& options) 
{  
     return newTwoLevelIterator(index_iter, block_function, arg, options);  
}  
```


这里有一个函数指针BlockFunction，类型为：

```
typedef Iterator* (*BlockFunction)(void*, const ReadOptions&, constSlice&);
```

**为什么叫TwoLevelIterator呢**，下面就来看看。

#### 6.6.2 TwoLevelIterator

它也是**Iterator的子类**，之所以叫two level应该是不仅可以迭代其中存储的对象，它还接受了一个函数**BlockFunction**，可以遍历存储的对象，可见它是专门为**Table定制**的。
我们已经知道各种Block的**存储格式**都是**相同**的，但是各自block data存储的**k/v**又**互不相同**，于是我们就需要一个途径，能够在使用同一个方式**遍历**不同的block时，又能**解析**这些k/v。这就是**BlockFunction**，它又返回了一个针对block data的Iterator。**Block和block data**存储的k/v对的key是统一的。
先来看类的主要成员变量：

```
BlockFunction block_function_; // block操作函数  
void* arg_;                    // BlockFunction的自定义参数  
const ReadOptions options_;    // BlockFunction的read option参数  
Status status_;                // 当前状态  
IteratorWrapper index_iter_;   // 遍历block的迭代器  
IteratorWrapper data_iter_;    // May be NULL-遍历block data的迭代器  
// 如果data_iter_ != NULL，data_block_handle_保存的是传递给  
// block_function_的index value，以用来创建data_iter_  
std::string data_block_handle_; 
```

下面**分析一下对于Iterator几个接口的实现。**

##### S1 对于其Key和Value接口都是返回的data_iter_对应的key和value：

```
virtual bool Valid() const 
{  
      return data_iter_.Valid();  
} 

virtual Slice key() const 
{   
       assert(Valid());  
       return data_iter_.key();  
}

virtual Slice value() const 
{  
       assert(Valid());  
       return data_iter_.value();  
} 
```

**S2 在分析Seek系函数之前**，有必要先了解下面这几个函数的用途。

```
void InitDataBlock();  
void SetDataIterator(Iterator*data_iter); 
//设置date_iter_ = data_iter  
voidSkipEmptyDataBlocksForward();  
voidSkipEmptyDataBlocksBackward(); 
```

###### S2.1首先是InitDataBlock()，它是根据index_iter来初始化data_iter，当定位到新的block时，需要更新data Iterator，指向该block中k/v对的合适位置，函数如下：

```
if (!index_iter_.Valid()) SetDataIterator(NULL);
// index_iter非法  
else 
{  
      Slice handle =index_iter_.value();  
      if (data_iter_.iter() != NULL&& handle.compare(data_block_handle_) == 0)
      {  
           //data_iter已经在该block data上了，无须改变  
         } 
      else 
      { 
           // 根据handle数据定位data iter  
           Iterator* iter =(*block_function_)(arg_, options_, handle);  
           data_block_handle_.assign(handle.data(), handle.size());  
           SetDataIterator(iter);  
      }  
}
```

###### S2.2 SkipEmptyDataBlocksForward，向前跳过空的datablock，函数实现如下：

```
while (data_iter_.iter() == NULL|| !data_iter_.Valid()) 
{   
      // 跳到下一个block  
      if (!index_iter_.Valid()) 
      { 
           // 如果index iter非法，设置data iteration为NULL  
           SetDataIterator(NULL);  
           return;  
         }  
      index_iter_.Next();  
      InitDataBlock();  
      if (data_iter_.iter() != NULL)data_iter_.SeekToFirst();
      // 跳转到开始  
}  
```

###### S2.3 SkipEmptyDataBlocksBackward，向后跳过空的datablock，函数实现如下：

```
while (data_iter_.iter() == NULL|| !data_iter_.Valid()) 
{ 
     // 跳到前一个block  
     if (!index_iter_.Valid()) 
     { 
          // 如果index iter非法，设置data iteration为NULL  
          SetDataIterator(NULL);  
          return;  
        }  
        index_iter_.Prev();  
      InitDataBlock();  
      if (data_iter_.iter() != NULL)data_iter_.SeekToLast();
      // 跳转到开始  
} 
```

##### S3 了解了几个跳转的辅助函数，再来看Seek系接口。

```
void TwoLevelIterator::Seek(const Slice& target) 
{  
    index_iter_.Seek(target);  
    InitDataBlock(); 
    // 根据index iter设置data iter  
    if (data_iter_.iter() != NULL)data_iter_.Seek(target); 
    // 调整data iter跳转到target  
    SkipEmptyDataBlocksForward(); 
    // 调整iter，跳过空的block  
}  

void TwoLevelIterator::SeekToFirst() 
{  
    index_iter_.SeekToFirst();  
    InitDataBlock();              // 根据index iter设置data iter  
    if (data_iter_.iter() != NULL)data_iter_.SeekToFirst();  
    SkipEmptyDataBlocksForward(); // 调整iter，跳过空的block  
} 

void TwoLevelIterator::SeekToLast() 
{  
    index_iter_.SeekToLast();  
    InitDataBlock();              // 根据index iter设置data iter  
    if (data_iter_.iter() != NULL)data_iter_.SeekToLast();  
    SkipEmptyDataBlocksBackward();// 调整iter，跳过空的block  
}

void TwoLevelIterator::Next() 
{  
     assert(Valid());  
     data_iter_.Next();  
     SkipEmptyDataBlocksForward(); // 调整iter，跳过空的block  
}  

void TwoLevelIterator::Prev()
{  
     assert(Valid());  
     data_iter_.Prev();  
     SkipEmptyDataBlocksBackward();// 调整iter，跳过空的block  
}  
```

#### 6.6.3 BlockReader()

上面**传递给twolevel Iterator的函数是Table::BlockReader函数**，声明如下：

```
static Iterator* Table::BlockReader(void* arg, const ReadOptions&options,
                                  constSlice& index_value);
```

它根据参数指明的**blockdata**，返回一个iterator对象，调用者就可以通过这个iterator对象遍历blockdata存储的k/v对，这其中用到了**LRUCache**。
函数实现逻辑如下：

##### S1 从参数中解析出BlockHandle对象，其中arg就是Table对象，index_value存储的是BlockHandle对象，读取Block的索引。

```
Table* table =reinterpret_cast<Table*>(arg);  
Block* block = NULL;  
Cache::Handle* cache_handle =NULL;  
BlockHandle handle;  
Slice input = index_value;  
Status s =handle.DecodeFrom(&input);
```

##### S2 根据block handle，首先尝试从cache中直接取出block，不在cache中则调用ReadBlock从文件读取，读取成功后，根据option尝试将block加入到LRU cache中。并在Insert的时候注册了释放函数DeleteCachedBlock。

```
Cache* block_cache =table->rep_->options.block_cache;  
BlockContents contents;  
if (block_cache != NULL)
{  
     char cache_key_buffer[16]; 
     // cache key的格式为table.cache_id + offset  
     EncodeFixed64(cache_key_buffer, table->rep_->cache_id);  
     EncodeFixed64(cache_key_buffer+8, handle.offset());  
     Slice key(cache_key_buffer,sizeof(cache_key_buffer));  
     cache_handle =block_cache->Lookup(key); // 尝试从LRU cache中查找  
     if (cache_handle != NULL)
     { 
          // 找到则直接取值  
          block =reinterpret_cast<Block*>(block_cache->Value(cache_handle));  
     } 
     else 
     { 
          // 否则直接从文件读取  
          s =ReadBlock(table->rep_->file, 
                       options, handle, &contents);  
          if (s.ok()) 
          {  
               block = new Block(contents);  
               if (contents.cachable&& options.fill_cache) 
               // 尝试加到cache中  
              cache_handle =block_cache->Insert(key, block,block->size(), &DeleteCachedBlock);  
         }  
     }  
} 
else
{  
     s =ReadBlock(table->rep_->file, options, handle, &contents);  
     if (s.ok()) block = newBlock(contents);  
}
```

##### S3 如果读取到了block，调用Block::NewIterator接口创建Iterator，如果cache handle为NULL，则注册DeleteBlock，否则注册ReleaseBlock，事后清理。

```
Iterator* iter;  
if (block != NULL)
{  
    iter =block->NewIterator(table->rep_->options.comparator);  
    if (cache_handle == NULL)  iter->RegisterCleanup(&DeleteBlock,block, NULL);  
    else iter->RegisterCleanup(&ReleaseBlock,block_cache, cache_handle);  
} 
else iter = NewErrorIterator(s); 
```


处理结束，最后**返回iter**。这里简单列下这几个**静态函数**，都很简单：

```
static void DeleteBlock(void* arg, void* ignored) 
{ 
     deletereinterpret_cast<Block*>(arg);
} 

static void DeleteCachedBlock(const Slice& key, void* value)   
{  
      Block* block =reinterpret_cast<Block*>(value);  
      delete block;  
} 

static void ReleaseBlock(void* arg, void* h) 
{  
      Cache* cache =reinterpret_cast<Cache*>(arg);  
      Cache::Handle* handle =reinterpret_cast<Cache::Handle*>(h);  
      cache->Release(handle);  
} 
```

### 6.7 定位key

这里并不是精确的定位，而是在Table中找到**第一个>=指定key的k/v对**，然后返回其value在sstable文件中的偏移。也是Table类的一个接口：

```
uint64_t ApproximateOffsetOf(const Slice& key) const;
```

函数实现比较简单：

##### S1 调用Block::Iter的Seek函数定位

```
Iterator* index_iter=rep_->index_block->NewIterator(rep_->options.comparator);  
index_iter->Seek(key);  
uint64_t result; 
```

##### S2 如果index_iter是合法的值，并且Decode成功，返回结果offset。

```
BlockHandle handle;  
handle.DecodeFrom(&index_iter->value());  
result = handle.offset(); 
```

##### S3 其它情况，设置result为rep_->metaindex_handle.offset()，metaindex的偏移在文件结尾附近。

### 6.8 获取Key—InternalGet()

**InternalGet**，这是为TableCache开的一个口子。这是一个**private函数**，声明为：

```
Status Table::InternalGet(const ReadOptions& options, constSlice& k,
                          void*arg, void (*saver)(void*, const Slice&, const Slice&))
```

其中又有函数指针，在找到数据后，就调用传入的**函数指针save**r执行调用者的自定义处理逻辑，并且**TableCache**可能会做缓存。
函数逻辑如下：

##### S1 首先根据传入的key定位数据，这需要indexblock的Iterator。

```
Iterator* iiter =rep_->index_block->NewIterator(rep_->options.comparator);  
iiter->Seek(k); 
```

##### S2 如果key是合法的，取出其filter指针，如果使用了filter，则检查key是否存在，这可以快速判断，提升效率。

```
Status s;  
Slice handle_value =iiter->value();  
FilterBlockReader* filter = rep_->filter;  
BlockHandle handle;  
if (filter != NULL && handle.DecodeFrom(&handle_value).ok() && !filter->KeyMayMatch(handle.offset(),k)) 
{ 
    // key不存在  
} 
else
{
    // 否则就要读取block，并查找其k/v对  
    Slice handle = iiter->value();  
    Iterator* block_iter =BlockReader(this, options, iiter->value());  
    block_iter->Seek(k);  
    if (block_iter->Valid())(*saver)(arg, block_iter->key(), block_iter->value());  
    s = block_iter->status();  
    delete block_iter;  
}
```

##### S3 最后返回结果，删除临时变量。

```
if (s.ok()) s =iiter->status();  
delete iiter;  
return s; 
```


随着有关**sstable文件读取**的结束，sstable的源码也就分析完了，其中我们还遗漏了一些功课要做，那就是**Filter和TableCache**部分。
