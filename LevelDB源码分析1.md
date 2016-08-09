LevelDB源码分析1
reference:
<br>http://blog.csdn.net/sparkliang/article/details/8567602

#### 1、一些约定
1.1 字节序
Leveldb对于数字的存储是little-endian，把int32或者int64转换为char*的函数中，是按照先低位，后高位的顺序存放，也就是litte-endian。

1.2 VarInt
把一个int32或者int64格式化到字符串中，除了上面说的little-endian字节序外，大部分还是变长存储的，也就是VarInt。对于VarInt，每byte的有效存储是7bit的，用最高的8bit位来表示是否结束，如果是1就表示后面还有一个byte的数字，否则表示结束。

1.3 字符比较
是基于unsigned char的，而非char。

#### 2、基本数据结构
2.1 Slice
Leveldb中的基本数据结构，它包括length和一个指向外部字节数组的指针。和string一样，允许字符串中包含’\0’。
提供一些基本接口，可以把const char*和string转换为Slice；把Slice转换为string，取得数据指针const char*。

2.2 Status
Leveldb 中的返回状态，将错误号和错误信息封装成Status类，统一进行处理。并定义了几种具体的返回状态，如成功或者文件不存在等。
为了节省空间Status并没有用std::string来存储错误信息，而是将返回码(code), 错误信息message及长度打包存储于一个字符串数组中。
成功状态OK 是NULL state_，否则state_ 是一个包含如下信息的数组:  
state_[0..3] == 消息message长度 
state_[4]    == 消息code
state_[5..]  ==消息message 

2.3 Arena
