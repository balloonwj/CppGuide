# leveldb源码分析13

本系列《leveldb源码分析》共有22篇文章，这是第十三篇

## 8.FilterPolicy&Bloom之二

### 8.5 构建FilterBlock

#### 8.5.1 FilterBlockBuilder

了解了filter机制，现在来看看**filter block的构建**，这就是类FilterBlockBuilder。它为指定的table构建所有的filter，结果是一个string字符串，并作为一个block存放在table中。它有三个函数接口：

```
// 开始构建新的filter block，TableBuilder在构造函数和Flush中调用  

void StartBlock(uint64_tblock_offset);  

// 添加key，TableBuilder每次向data block中加入key时调用  

void AddKey(const Slice&key);  

// 结束构建，TableBuilder在结束对table的构建时调用  

Slice Finish();    
```

**FilterBlockBuilder的构建顺序**必须满足如下范式：`(StartBlock AddKey*)* Finish`，显然这和前面讲过的BlockBuilder有所不同。
其成员变量有：

```
const FilterPolicy* policy_; // filter类型，构造函数参数指定  

std::string keys_;          //Flattened key contents  

std::vector<size_t> start_; // 各key在keys_中的位置  

std::string result_;        // 当前计算出的filter data  

std::vector<uint32_t>filter_offsets_; // 各个filter在result_中的位置  

std::vector<Slice> tmp_keys_;// policy_->CreateFilter()参数   
```

前面说过base是2KB，这对应两个常量`kFilterBase =11, kFilterBase =(1<<kFilterBaseLg)；`其实从后面的实现来看tmp_keys_完全不必作为成员变量，直接作为函数GenerateFilter()的栈变量就可以。下面就分别分析三个函数接口。

#### 8.5.2 FilterBlockBuilder::StartBlock()

它根据参数block_offset计算出filter index，然后循环调用GenerateFilter生产新的Filter。

```
uint64_t filter_index =(block_offset / kFilterBase);  

assert(filter_index >=filter_offsets_.size());  

while (filter_index >filter_offsets_.size()) GenerateFilter();  
```

我们来到GenerateFilter这个函数，看看它的逻辑。

```
//S1 如果filter中key个数为0，则直接压入result_.size()并返回  

  const size_t num_keys =start_.size();  

  if (num_keys == 0) { // there are no keys for this filter  

    filter_offsets_.push_back(result_.size()); //result_.size()应该是0  

    return;  

  }  

//S2 从key创建临时key list，根据key的序列字符串kyes_和各key在keys_
//中的开始位置start_依次提取出key。  

  start_.push_back(keys_.size());  // Simplify lengthcomputation  

  tmp_keys_.resize(num_keys);  

  for (size_t i = 0; i <num_keys; i++) {  

    const char* base =keys_.data() + start_[i]; // 开始指针  

    size_t length = start_[i+1] -start_[i]; // 长度  

    tmp_keys_[i] = Slice(base,length);  

  }  

//S3 为当前的key集合生产filter，并append到result_  

filter_offsets_.push_back(result_.size());  

policy_->CreateFilter(&tmp_keys_[0], num_keys, &result_);  

//S4 清空，重置状态  

tmp_keys_.clear();  

keys_.clear();  

start_.clear();  
```

#### 8.5.3 FilterBlockBuilder::AddKey()

这个接口很简单，就是把key添加到key_中，并在start_中记录位置。

```
Slice k = key;  

start_.push_back(keys_.size());  

keys_.append(k.data(),k.size());  
```

#### 8.5.4 FilterBlockBuilder::Finish()

调用这个函数说明整个table的data block已经构建完了，可以生产最终的filter block了，在TableBuilder::Finish函数中被调用，向sstable写入meta block。
函数逻辑为：

```
//S1 如果start_数字不空，把为的key列表生产filter  

  if (!start_.empty()) GenerateFilter();  

//S2 从0开始顺序存储各filter的偏移值，见filter block data的数据格式。  

  const uint32_t array_offset =result_.size();  

  for (size_t i = 0; i < filter_offsets_.size();i++) {  

    PutFixed32(&result_,filter_offsets_[i]);  

  }  

//S3 最后是filter个数，和shift常量（11），并返回结果  

  PutFixed32(&result_,array_offset);  

  result_.push_back(kFilterBaseLg);  // Save encoding parameter in result  

  return Slice(result_);  
```

#### 8.5.5 简单示例

让我们根据TableBuilder对FilterBlockBuilder接口的调用范式：
(StartBlock AddKey*)* Finish以及上面的函数实现，结合一个简单例子看看**leveldb是如何为data block创建filter block（也就是meta block）的。**
考虑两个datablock，在sstable的范围分别是：`Block 1 [0, 7KB-1], Block 2 [7KB, 14.1KB]`

- S1 首先TableBuilder为Block 1调用FilterBlockBuilder::StartBlock(0)，该函数直接返回；

- S2 然后依次向Block 1加入k/v，其中会调用FilterBlockBuilder::AddKey，FilterBlockBuilder记录这些key。

- S3 下一次TableBuilder添加k/v时，例行检查发现Block 1的大小超过设置，则执行Flush操作，Flush操作在写入Block 1后，开始准备Block 2并更新block offset=7KB，最后调用FilterBlockBuilder::StartBlock(7KB)，开始为Block 2构建Filter。

- S4 在FilterBlockBuilder::StartBlock(7KB)中，计算出filter index = 3，触发3次GenerateFilter函数，为Block 1添加的那些key列表创建filter，其中第2、3次循环创建的是空filter。此时filter的结构如图8.5-1所示。![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

    图8.5-1

    在StartBlock(7KB)时会向filter的偏移数组filter_offsets_压入两个包含空key set的元素，filter_offsets_[1]和filter_offsets_[2]，它们的值都等于7KB-1。

- S5Block 2构建结束，TableBuilder调用Finish结束table的构建，这会再次触发Flush操作，在写入Block 2后，为Block 2的key创建filter。最终的filter如图8.5-2所示。![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

    图8.5-2

- 这里如果Block 1的范围是[0, 1.8KB-1]，Block 2从1.8KB开始，那么Block 2将会和Block 1共用一个filter，它们的filter都被生成到filter 0中。
    当然在TableBuilder构建表时，Block的大小是根据参数配置的，也是基本均匀的。

### 8.6 读取FilterBlock

#### 8.6.1 FilterBlockReader

FilterBlock的读取操作在FilterBlockReader类中，它的主要**功能**是根据传入的FilterPolicy和filter，进行key的匹配查找。
它有如下的几个成员变量：

```
const FilterPolicy* policy_; // filter策略  

const char* data_;           // filter data指针 (at block-start)  

const char* offset_;        // offset array的开始地址 (at block-end)  

size_t num_;                // offsetarray元素个数  

size_t base_lg_;            // 还记得kFilterBaseLg吗  
```

Filter策略和filter block内容都由构造函数传入。一个接口函数，就是key的批判查找：

```
bool KeyMayMatch(uint64_t block_offset, const Slice& key);
```

#### 8.6.2 构造

在构造函数中，根据存储格式解析出偏移数组开始指针、个数等信息。

```
FilterBlockReader::FilterBlockReader(const FilterPolicy* policy, 
                                     constSlice& contents)  

    : policy_(policy),data_(NULL), offset_(NULL), num_(0), base_lg_(0) {  

  size_t n = contents.size();  

  if (n < 5) return;  // 1 byte forbase_lg_ and 4 for start of offset array  

  base_lg_ = contents[n-1]; // 最后1byte存的是base  

  uint32_t last_word =DecodeFixed32(contents.data() + n - 5); //偏移数组的位置  

  if (last_word > n - 5)return;  

  data_ = contents.data();  

  offset_ = data_ + last_word; // 偏移数组开始指针  

  num_ = (n - 5 - last_word) / 4; // 计算出filter个数  
```

#### 8.6.3 查找

查找函数传入两个参数

- @block_offset是查找data block在sstable中的偏移，Filter根据此偏移计算filter的编号；
- @key是查找的key。
    声明如下：
    `bool FilterBlockReader::KeyMayMatch(uint64_t block_offset, constSlice& key)`
    它**首先**计算出filterindex，**根据**index解析出filter的range，**如果**是合法的range，就从data\_中取出filter，调用policy_做key的匹配查询。函数实现：

```
uint64_t index = block_offset>> base_lg_; // 计算出filter index  

if (index < num_) {  
  // 解析出filter的range  
  uint32_t start =DecodeFixed32(offset_ + index*4);

  uint32_t limit =DecodeFixed32(offset_ + index*4 + 4);  

  if (start <= limit&& limit <= (offset_ - data_)) {  

    Slice filter = Slice(data_ +start, limit - start); // 根据range得到filter  

    returnpolicy_->KeyMayMatch(key, filter);  

  } else if (start == limit) {  

    return false;  // 空filter不匹配任何key  

  }  

}  

return true; // 当匹配处理  
```

至此，FilterPolicy和Bloom就分析完了。
