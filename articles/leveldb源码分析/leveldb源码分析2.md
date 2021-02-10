# leveldb源码分析2

本系列《leveldb源码分析》共有22篇文章，这是第二篇。

### 3.Int Coding

轻松一刻，前面约定中讲过Leveldb使用了很多VarInt型编码，典型的如后面将涉及到的各种key。其中的**编码、解码函数分为VarInt和FixedInt两种**。int32和int64操作都是类似的。

#### 3.1 Eecode

首先是**FixedInt编码**，直接上代码，很简单明了。

```
void EncodeFixed32(char* buf, uint32_t value)
{
  if (port::kLittleEndian) {
    memcpy(buf, &value,sizeof(value));
  } else {
    buf[0] = value & 0xff;
    buf[1] = (value >> 8)& 0xff;
    buf[2] = (value >> 16)& 0xff;
    buf[3] = (value >> 24)& 0xff;
  }
}
```

下面是**VarInt编码**，int32和int64格式，代码如下，有效位是7bit的，因此把uint32按7bit分割，对unsigned char赋值时，超出0xFF会自动截断，因此直接*(ptr++) = v|B即可，不需要再把(v|B)与0xFF作&操作。

```
char* EncodeVarint32(char* dst, uint32_t v)
{
  unsigned char* ptr =reinterpret_cast<unsigned char*>(dst);
  static const int B = 128;
  if (v < (1<<7)) {
    *(ptr++) = v;
  } else if (v < (1<<14)){
    *(ptr++) = v | B;
    *(ptr++) = v>>7;
  } else if (v < (1<<21)){
    *(ptr++) = v | B;
    *(ptr++) = (v>>7) | B;
    *(ptr++) = v>>14;
  } else if (v < (1<<28)){
    *(ptr++) = v | B;
    *(ptr++) = (v>>7) | B;
    *(ptr++) = (v>>14) | B;
    *(ptr++) = v>>21;
  } else {
    *(ptr++) = v | B;
    *(ptr++) = (v>>7) | B;
    *(ptr++) = (v>>14) | B;
    *(ptr++) = (v>>21) | B;
    *(ptr++) = v>>28;
  }
  return reinterpret_cast<char*>(ptr);
}

// 对于uint64，直接循环
char* EncodeVarint64(char* dst, uint64_t v) {
  static const int B = 128;
  unsigned char* ptr =reinterpret_cast<unsigned char*>(dst);
  while (v >= B) {
    *(ptr++) = (v & (B-1)) |B;
    v >>= 7;
  }
  *(ptr++) =static_cast<unsigned char>(v);
  returnreinterpret_cast<char*>(ptr);
}
```

#### 3.2 Decode

**Fixed Int的Decode**，操作，代码：

```
inline uint32_t DecodeFixed32(const char* ptr)
{
  if (port::kLittleEndian) {
    uint32_t result;
    // gcc optimizes this to a plain load
    memcpy(&result, ptr,sizeof(result));
    return result;
  } else {
    return((static_cast<uint32_t>(static_cast<unsigned char>(ptr[0])))
        |(static_cast<uint32_t>(static_cast<unsigned char>(ptr[1])) <<8)
        | (static_cast<uint32_t>(static_cast<unsignedchar>(ptr[2])) << 16)
        |(static_cast<uint32_t>(static_cast<unsigned char>(ptr[3])) <<24));
  }
}
```

再来看看**VarInt的解码**，很简单，依次读取1byte，直到最高位为0的byte结束，取低7bit，作(<<7)移位操作组合成Int。看代码：

```
const char* GetVarint32Ptr(const char* p,
                           const char* limit, 
                           uint32_t* value)
{
  if (p < limit) {
    uint32_t result =*(reinterpret_cast<const unsigned char*>(p));
    if ((result & 128) == 0) {
      *value = result;
      return p + 1;
    }
  }
  return GetVarint32PtrFallback(p,limit, value);
}

const char* GetVarint32PtrFallback(const char* p,
                                   const char* limit,
                                   uint32_t* value)
{
  uint32_t result = 0;
  for (uint32_t shift = 0; shift<= 28 && p < limit; shift += 7) {
    uint32_t byte =*(reinterpret_cast<const unsigned char*>(p));
    p++;
    if (byte & 128) { // More bytes are present
      result |= ((byte & 127)<< shift);
    } else {
      result |= (byte <<shift);
      *value = result;
      returnreinterpret_cast<const char*>(p);
    }
  }
  return NULL;
}
```
