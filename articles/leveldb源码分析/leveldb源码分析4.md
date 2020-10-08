# leveldb源码分析4

本系列《leveldb源码分析》共有22篇文章，这是第四篇

## 4.Memtable之2 

### 4.6 Comparator

弄清楚了key，接下来就要看看key的使用了，先从**Comparator**开始分析。首先Comparator是一个抽象类，导出了几个接口。

其中**Name()**和**Compare()**接口都很明了，另外的两个Find xxx接口都有什么功能呢，直接看程序注释：

```
//Advanced functions: these are used to reduce the space requirements 
//for internal data structures like index blocks.                      
// 这两个函数：用于减少像index blocks这样的内部数据结构占用的空间
// 其中的*start和*key参数都是IN OUT的。
//If *start < limit, changes *start to a short string in [start,limit).
//Simple comparator implementations may return with *start unchanged,
//i.e., an implementation of this method that does nothing is correct.
// 这个函数的作用就是：如果*start < limit，就在[startlimit,)中找到一个
// 短字符串，并赋给*start返回
// 简单的comparator实现可能不改变*start，这也是正确的
virtual void FindShortestSeparator(std::string* start, 
                                   const Slice& limit) const = 0;
//Changes *key to a short string >= *key.
//Simple comparator implementations may return with *key unchanged,
//i.e., an implementation of this method that does nothing is correct. 
//这个函数的作用就是：找一个>= *key的短字符串
//简单的comparator实现可能不改变*key，这也是正确的
virtual void FindShortSuccessor(std::string* key) const = 0;
```

其中的**实现类有两个**，一个是**内置的BytewiseComparatorImpl**，另一个是**InternalKeyComparator**。下面分别来分析。

#### 4.6.1 BytewiseComparatorImpl

首先是重载的Name和比较函数，比较函数如其名，就是字符串比较，如下：

```
virtual const char* Name() const {return"leveldb.BytewiseComparator";}
virtual int Compare(const Slice& a, const Slice& b) const {return a.compare(b);}
```

再来看看Byte wise的comparator是如何实现**FindShortestSeparator()**的，没什么特别的，代码 + 注释如下：

```
virtual void FindShortestSeparator(std::string* start, 
                                   onst Slice& limit) const 
{
  // 首先计算共同前缀字符串的长度
  size_t min_length = std::min(start->size(), limit.size());
  size_t diff_index = 0;
  while ((diff_index < min_length) &&
        ((*start)[diff_index] == limit[diff_index]))
  {
     diff_index++;
  }
  if (diff_index >= min_length) 
  {
     // 说明*start是limit的前缀，或者反之，此时不作修改，直接返回
  } 
  else 
  {
     // 尝试执行字符start[diff_index]++，
        设置start长度为diff_index+1，并返回
     // ++条件：字符< oxff 并且字符+1 < limit上该index的字符
     uint8_t diff_byte = static_cast<uint8_t>((*start)[diff_index]);
     if (diff_byte < static_cast<uint8_t>(0xff) &&
         diff_byte + 1 < static_cast<uint8_t>(limit[diff_index])) 
     {
         (*start)[diff_index]++;
         start->resize(diff_index + 1);
          assert(Compare(*start, limit) < 0);
     }
  }
}
```

最后是FindShortSuccessor()，这个更简单了，代码+注释如下：

```
virtual void FindShortSuccessor(std::string* key) const 
{
    // 找到第一个可以++的字符，执行++后，截断字符串；
    // 如果找不到说明*key的字符都是0xff啊，那就不作修改，直接返回
    size_t n = key->size();
    for (size_t i = 0; i < n; i++) 
    {
        const uint8_t byte = (*key)[i];
        if (byte != static_cast<uint8_t>(0xff)) 
        {
            (*key)[i] = byte + 1;
            key->resize(i+1);
            return;
        }
     }
}
```

Leveldb内建的基于**Byte wise**的comparator类就这么多内容了，下面再来看看InternalKeyComparator。

#### 4.6.2 InternalKeyComparator

从上面对Internal Key的讨论可知，由于它是由**user key和sequence number和value type组合而成**的，因此它还需要user key的比较，所以InternalKeyComparator有一个Comparator* user_comparator_成员，用于**user key**的比较。

在leveldb中的名字为："leveldb.InternalKeyComparator"，下面来看看比较函数：

```
Compare(const Slice& akey, const Slice& bkey)
```

代码很简单，其比较逻辑是：

- S1 首先比较user key，基于用户设置的comparator，如果**user key不相等**就直接**返回比较**,**否则**执行进入S2
- S2 取出8字节的sequence number | value type，如果akey的 **> bkey**的则返回**-1**，如果akey的**<bkey**的返回**1**，**相等**返回**0**

由此可见其排序比较依据依次是：

1. 首先根据user key按升序排列
2. 然后根据sequence number按降序排列
3. 最后根据value type按降序排列

虽然比较时value type并不重要，因为sequence number是唯一的，但是直接取出8byte的sequence number | value type，然后做比较**更方便**，不需要再次移位提取出7byte的sequence number，又何乐而不为呢。这也是把value type安排在低7byte的好处吧，排序的两个依据就是**user key和sequence number**。

接下来就该看看其**FindShortestSeparator()函数**实现了，该函数取出Internal Key中的user key字段，根据user指定的comparator找到并替换start，如果start被替换了，就用**新的start更新Internal Key**，并使用最大的**sequence number**。否则保持不变。

函数声明：

```
void InternalKeyComparator::FindShortestSeparator(std::string* start, const Slice& limit) const;
```

函数实现：

```
// 尝试更新user key，基于指定的user comparator
Slice user_start = ExtractUserKey(*start);
Slice user_limit = ExtractUserKey(limit);
std::string tmp(user_start.data(), user_start.size());
user_comparator_->FindShortestSeparator(&tmp, user_limit);
if(tmp.size()<user_start.size()&&
   user_comparator_->Compare(user_start, tmp)<0)
{
    // user key在物理上长度变短了，但其逻辑值变大了.生产新的*start时，
    // 使用最大的sequence number，以保证排在相同user key记录序列的第一个
    PutFixed64(&tmp, PackSequenceAndType(kMaxSequenceNumber,
                                       kValueTypeForSeek));
    assert(this->Compare(*start, tmp) < 0);
    assert(this->Compare(tmp, limit) < 0);
    start->swap(tmp);
}
```

接下来是**FindShortSuccessor(std::string\* key)**函数，该函数取出Internal Key中的**user key**字段，根据user指定的**comparator**找到并替换key，如果key被替换了，就用新的key更新**Internal Key**，并使用最大的**sequence number**。否则保持不变。实现逻辑如下：

```
Slice user_key = ExtractUserKey(*key);
// 尝试更新user key，基于指定的user comparator
std::string tmp(user_key.data(), user_key.size());
user_comparator_->FindShortSuccessor(&tmp);
if(tmp.size()<user_key.size() && 
   user_comparator_->Compare(user_key, tmp)<0)
{
   // user key在物理上长度变短了，但其逻辑值变大了.生产新的*start时，
   // 使用最大的sequence number，以保证排在相同user key记录序列的第一个
   PutFixed64(&tmp, PackSequenceAndType(kMaxSequenceNumber,
                                      kValueTypeForSeek));
   assert(this->Compare(*key, tmp) < 0);
   key->swap(tmp);
}
```



### 4.7 Memtable::Insert()

把相关的**Key和Key Comparator**都弄清楚后，是时候分析memtable本身了。首先是向memtable**插入记录的接**口，函数原型如下：

```
void Add(SequenceNumber seq, ValueType type, const Slice& key, const Slice& value);
```

代码实现如下：

```
// KV entry字符串有下面4部分连接而成
//key_size     : varint32 of internal_key.size()
//key bytes    : char[internal_key.size()]
//value_size   : varint32 of value.size()
//  value bytes  : char[value.size()]
size_t key_size = key.size();
size_t val_size = value.size();
size_t internal_key_size = key_size + 8;
const size_t encoded_len = VarintLength(internal_key_size) +
                           internal_key_size +
                           VarintLength(val_size) + val_size;
char* buf = arena_.Allocate(encoded_len);
char* p = EncodeVarint32(buf, internal_key_size);
memcpy(p, key.data(), key_size);
p += key_size;
EncodeFixed64(p, (s << 8) | type);
p += 8;
p = EncodeVarint32(p, val_size);
memcpy(p, value.data(), val_size);
assert((p + val_size) - buf == encoded_len);
able_.Insert(buf);
```

根据代码，我们可以分析出KV记录在skip list的**存储格式**等信息，首先总长度为：

```
VarInt(Internal Key size) len + internal key size + VarInt(value) len + value size
```

它们的相互衔接也就是KV的存储格式：

```
| VarInt(Internal Key size) len | internal key |VarInt(value) len |value|
```

其中前面说过：

```
internal key = |user key |sequence number |type |
Internal key size = key size + 8
```

### 4.8 Memtable::Get()

Memtable的查找接口，根据一个**LookupKey**找到响应的记录，函数声明：

```
bool MemTable::Get(const LookupKey& key, std::string* value, Status* s)
```

函数实现如下：

```
Slice memkey = key.memtable_key();
Table::Iterator iter(&table_);
iter.Seek(memkey.data()); 
// seek到value>= memkey.data()的第一个记录
if (iter.Valid()) 
{
    // 这里不需要再检查sequence number了，因为Seek()已经跳过了所有
    // 值更大的sequence number了
    const char* entry = iter.key();
    uint32_t key_length;
    const char* key_ptr = GetVarint32Ptr(entry, entry+5, 
                                       &key_length);
    // 比较user key是否相同，key_ptr开始的len(internal key) -8 byte是user key
    if (comparator_.comparator.user_comparator()->Compare
      (Slice(key_ptr, key_length - 8), key.user_key()) == 0) 
    {
        // len(internal key)的后8byte是 |sequence number | value type|
        const uint64_t tag = DecodeFixed64(key_ptr + key_length - 8);
        switch (static_cast<ValueType>(tag & 0xff))
        {
        case kTypeValue: 
        {
           // 只取出value
           Slice v = GetLengthPrefixedSlice(key_ptr + key_length);
           value->assign(v.data(), v.size());
           return true;
        }
       case kTypeDeletion:
       *s = Status::NotFound(Slice());
       return true;
      }
   }
}
return false;
```

这段代码，主要就是一个**Seek函数**，根据传入的LookupmKey得到在emtable中存储的key，然后调用Skip list::Iterator的Seek函数查找。Seek**直接调用**Skip list的FindGreaterOrEqual(key)接口，返回**大于等于key的Iterator**。然后取出user key判断时候和传入的user key相同，如果**相同**则**取出value**，如果记录的Value Type为kTypeDeletion，返回Status::NotFound(Slice())。

### 4.9 小结

Memtable到此就分析完毕了，本质上就是一个有序的Skip list，排序基于user key的sequence number，其**排序比较依据**依次是：

1. **首先根据user key按升序排列**
2. **然后根据sequence number按降序排列**
3. **最后根据value type按降序排列（这个其实无关紧要）**
