
leveldb有自己自带的字符串string,而不是用C++原生的string.在leveldb中，是用Slice类来实现的。

看看这个类的成员：有两个,data_和size_.

```
 private:
  const char* data_;		//数据：是一个const char*
  size_t size_;				//字符串的长度
```

一些构造函数，可以用字符串和string来初始化一个Slice

```
public:
  // Create an empty slice.
  //创建一个空的字符串，默认构造函数。
  Slice() : data_(""), size_(0) { }

  // Create a slice that refers to d[0,n-1].
  //用字符串指针 d和字符串长度初始化一个Slice
  Slice(const char* d, size_t n) : data_(d), size_(n) { }

  // Create a slice that refers to the contents of "s"
  //用string初始化Slice
  Slice(const std::string& s) : data_(s.data()), size_(s.size()) { }

  // Create a slice that refers to s[0,strlen(s)-1]
  //用字符串指针初始化Slice
  Slice(const char* s) : data_(s), size_(strlen(s)) { }

```

其他的一些操作

```
  // Return true iff the length of the referenced data is zero
  //判断是否为空
  bool empty() const { return size_ == 0; }

  // Return the ith byte in the referenced data.
  // REQUIRES: n < size()
  //重载[]操作符，直接通过slice[n]获取第n个字符
  char operator[](size_t n) const {
    assert(n < size());
    return data_[n];
  }

  // Change this slice to refer to an empty array
  //清空一个Slice
  void clear() { data_ = ""; size_ = 0; }

  // Drop the first "n" bytes from this slice.
  //删除前面的n个字符,data指针向前移动n，size减小n
  void remove_prefix(size_t n) {
    assert(n <= size());
    data_ += n;
    size_ -= n;
  }
```

其他一些函数：
```
//判断是不是相等：字符串个数相同，并且每一个字符也相同
inline bool operator==(const Slice& x, const Slice& y) {
  return ((x.size() == y.size()) &&
          (memcmp(x.data(), y.data(), x.size()) == 0));
}

inline bool operator!=(const Slice& x, const Slice& y) {
  return !(x == y);
}

inline int Slice::compare(const Slice& b) const {
  const int min_len = (size_ < b.size_) ? size_ : b.size_;
  int r = memcmp(data_, b.data_, min_len);
  if (r == 0) {
    if (size_ < b.size_) r = -1;
    else if (size_ > b.size_) r = +1;
  }
  return r;
}
```
测试
```
#include <iostream>
#include <string.h>

#include "leveldb_slice.h"

using namespace std;

int main()
{
    const char* theChar = "xiaolan";
    leveldb::Slice s(theChar);

    cout << s.data() << endl;
    cout << s.size() << endl;
}

```