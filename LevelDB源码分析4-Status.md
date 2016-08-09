所在文件：status.cc和status.h
leveldb用一个Status类来表示函数执行的状态，用一个enum类型来表示。当没有错误时，state_为NULL，否则，会有错误码。
Status有一个cosnt char* state_，为了节省内存，分为3个部分使用，意义如下：
* state_[0..3]为消息的长度
* state_[4]为消息的类型
* state_[5..]为错误的消息

```
  const char* state_;

  enum Code {
    kOk = 0,
    kNotFound = 1,
    kCorruption = 2,
    kNotSupported = 3,
    kInvalidArgument = 4,
    kIOError = 5
  };
```

这些构造函数返回指定类型的Status
```
  // Return a success status.
  static Status OK() { return Status(); }

  // Return error status of an appropriate type.
  //yanke:构造函数：对于每一种Error，都有一个构造函数
  static Status NotFound(const Slice& msg, const Slice& msg2 = Slice()) {
    return Status(kNotFound, msg, msg2);
  }
  static Status Corruption(const Slice& msg, const Slice& msg2 = Slice()) {
    return Status(kCorruption, msg, msg2);
  }
  static Status NotSupported(const Slice& msg, const Slice& msg2 = Slice()) {
    return Status(kNotSupported, msg, msg2);
  }
  static Status InvalidArgument(const Slice& msg, const Slice& msg2 = Slice()) {
    return Status(kInvalidArgument, msg, msg2);
  }
  static Status IOError(const Slice& msg, const Slice& msg2 = Slice()) {
    return Status(kIOError, msg, msg2);
  }

```
这些都是调用下面的函数

```
//yanke:将Slice转为Status的构造函数
Status::Status(Code code, const Slice& msg, const Slice& msg2) {
  assert(code != kOk);
  const uint32_t len1 = msg.size();
  const uint32_t len2 = msg2.size();
  //判断第二个字符串是不是为0，如果不为0，则长度为len1+len2+2，为了加入": "
  //
  const uint32_t size = len1 + (len2 ? (2 + len2) : 0);
  char* result = new char[size + 5];//state_长度包含前5个字符
  memcpy(result, &size, sizeof(size));//首先存入长度
  result[4] = static_cast<char>(code);//然后存入错误码
  memcpy(result + 5, msg.data(), len1);//最后复制错误信息，分2部：第一步复制msg的内容
  if (len2) { //第二部：看mgs2是不是有信息，如果没有就不用管，如果有，要继续复制。
    result[5 + len1] = ':';//开始复制：先加入':'和' '
    result[6 + len1] = ' ';
    memcpy(result + 7 + len1, msg2.data(), len2);//复制msg2的内容。
  }
  state_ = result;
}
```

复制构造函数和复制操作运算符

```
inline Status::Status(const Status& s) {
  state_ = (s.state_ == NULL) ? NULL : CopyState(s.state_);
}
inline void Status::operator=(const Status& s) {
  // The following condition catches both aliasing (when this == &s),
  // and the common case where both s and *this are ok.
  if (state_ != s.state_) {
    delete[] state_;
    state_ = (s.state_ == NULL) ? NULL : CopyState(s.state_);
  }
}
```

它们都调用CopyState这个函数

```
const char* Status::CopyState(const char* state) {
  uint32_t size;
  memcpy(&size, state, sizeof(size));		//state的前4位保存长度信息，先提取出来放入size中
  char* result = new char[size + 5];		//为新的Status开辟空间。size+5为state的长度
  memcpy(result, state, size + 5);			//然后开始复制
  return result;
}
```
code函数的定义如下：先判断state_是否为空，如果为空，则返回kOK,否则取出第四个字节转为Code类型，然后返回

```
  Code code() const {
    return (state_ == NULL) ? kOk : static_cast<Code>(state_[4]);
  }
```

转为string。格式为：“错误码：” + “错误信息”

```
//yanke:Status转为string
std::string Status::ToString() const {
  if (state_ == NULL) {
    return "OK";
  } else {
    char tmp[30];
    const char* type;
    switch (code()) {
      case kOk:
        type = "OK";
        break;
      case kNotFound:
        type = "NotFound: ";
        break;
      case kCorruption:
        type = "Corruption: ";
        break;
      case kNotSupported:
        type = "Not implemented: ";
        break;
      case kInvalidArgument:
        type = "Invalid argument: ";
        break;
      case kIOError:
        type = "IO error: ";
        break;
      default:
        snprintf(tmp, sizeof(tmp), "Unknown code(%d): ",
                 static_cast<int>(code()));
        type = tmp;
        break;
    }
    std::string result(type);
    uint32_t length;
    memcpy(&length, state_, sizeof(length));
    result.append(state_ + 5, length);
    return result;
  }
}
```