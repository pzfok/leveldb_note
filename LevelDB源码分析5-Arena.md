
成员变量：
其中的blocks_是一个char*的vector,这个vector中每个元素都是指向一块内存(大小为4096，4K)

<img src="https://svplsg.bl3301.livefilestore.com/y3mHgmvq6IO5QuyzX9BxZTcUr_96oZE82mm-ca9JfoB3Fib542Ecw4Ioj9fuLQ_3EbKFj4O3qiXUMyv3070f7o_DANLHhQl7X9BHf4x_8jUz5pA-pDVg9CXKFzTzGx7eodKtZoQYqY8MVpFeKVXNxy8UuV8WN0d7-ZMl_hC8RjdSq4?width=504&height=470&cropmode=none" width="504" height="470" />

```
char* alloc_ptr_;//指向当前内存块剩余空间的首地址
size_t alloc_bytes_remaining_;//剩余的内存数
std::vector<char*> blocks_; //blocks_
size_t blocks_memory_;		//分配的总内存
```

构造函数和析构函数：
```
//构造函数：分配内存为0
Arena::Arena() {
  blocks_memory_ = 0;
  alloc_ptr_ = NULL;  // First allocation will allocate a block
  alloc_bytes_remaining_ = 0;
}

//析构函数：释放blocks_中每一个元素指向的内存
Arena::~Arena() {
  for (size_t i = 0; i < blocks_.size(); i++) {
    delete[] blocks_[i];
  }
}
```

内存分配

```
//内存分配函数：
//1、如果分配的内存小于等于alloc_bytes_remaining_内存，那么直接分配就行了
//2、否则调用AllocateFallback函数
inline char* Arena::Allocate(size_t bytes) {
  // The semantics of what to return are a bit messy if we allow
  // 0-byte allocations, so we disallow them here (we don't need
  // them for our internal use).
  assert(bytes > 0);
  if (bytes <= alloc_bytes_remaining_) {
    char* result = alloc_ptr_;
    alloc_ptr_ += bytes;
    alloc_bytes_remaining_ -= bytes;
    return result;
  }
  return AllocateFallback(bytes);
}

char* Arena::AllocateFallback(size_t bytes) {
	//如果比kBlockSize / 4还要多，直接给这个内存单独分配一个内存。
	//防止内存碎片
  if (bytes > kBlockSize / 4) {
    // Object is more than a quarter of our block size.  Allocate it separately
    // to avoid wasting too much space in leftover bytes.
    char* result = AllocateNewBlock(bytes);
    return result;
  }

  // We waste the remaining space in the current block.
  alloc_ptr_ = AllocateNewBlock(kBlockSize);	//直接分配4k的空间
  alloc_bytes_remaining_ = kBlockSize;			//然后这就是当前的块剩余的空间4K

  char* result = alloc_ptr_;					//这是新的空间
  alloc_ptr_ += bytes;							//这里，直接放弃了小于1K的剩余空间
  alloc_bytes_remaining_ -= bytes;				//新的剩余空间为kBlockSize-bytes
  return result;								//返回
}

//分配新的内存块函数：需求是block_bytes
char* Arena::AllocateNewBlock(size_t block_bytes) {
  char* result = new char[block_bytes];	//分配内存，block_bytes
  blocks_memory_ += block_bytes;//总的内存增加
  blocks_.push_back(result);//添加到内存指针数组中
  return result;
}
```

内存使用量：包含放弃的小于1K的空间

```
  size_t MemoryUsage() const {
    return blocks_memory_ + blocks_.capacity() * sizeof(char*);
  }
```

内存对齐分配

```
char* Arena::AllocateAligned(size_t bytes) {
	//根据void*的大小对齐
  const int align = sizeof(void*);    // We'll align to pointer size
  //align应该为2的幂，如果不为2的幂，说明出错了，检测到了如此地步。
  assert((align & (align-1)) == 0);   // Pointer size should be a power of 2
  //又用到&和%计算的指是一样的，但是&快
  size_t current_mod = reinterpret_cast<uintptr_t>(alloc_ptr_) & (align-1);
  //计算需要添加的字节数
  size_t slop = (current_mod == 0 ? 0 : align - current_mod);
  //因此内存对齐后需要的字节为bytes+slop
  size_t needed = bytes + slop;
  char* result;
  //如果需求小于声音的bytes，那么直接分配
  if (needed <= alloc_bytes_remaining_) {
    result = alloc_ptr_ + slop;
    alloc_ptr_ += needed;
    alloc_bytes_remaining_ -= needed;
  } else {
    // 否则调用AllocateFallback函数
    result = AllocateFallback(bytes);
  }
  assert((reinterpret_cast<uintptr_t>(result) & (align-1)) == 0);
  return result;
}
```