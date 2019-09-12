### Unicode

是计算机科学领域里的一项业界标准，包括字符集、编码方案等。Unicode并不是一套具体实现的字符集！而是一种标准。Unicode字符集定义了(几乎)所有字符到二进制序列的一种映射关系，也可以认为是字符到无符号整数的映射，比如，字符'a'对应49

### 字符集(字符编码)

字符集做的就是将字符唯一映射到一个二进制编码并具体实现它。如：ascii，utf-8，utf-16，utf-16be，utf-16le。值得注意的是，字符集并不一定就是unicode的实现，如ascii编码，仅仅可以表示英文字母与常用字符，不能表示汉字（ascii不能，但是可以通过某种扩展实现）

### 码点code point & 码元code unit

code point对应一个字符的完整二进制表示

code unit对应一种字符集中的最小编码单位，比如utf-8的最小编码单位是8bit，一个code unit就是就是8bit，utf-8可能需要1~6个code unit来表示一个字符，所以一个utf-8字符可能有1，2，3，4字节；而对于utf-16来说，一个code unit是16bit，即2字节，一个code point对于1个或2个code unit，即一个utf-16编码的字符可能是2字节或4字节。

### 小端模式Little Endian & 大端模式Big Endian

端endian的来源

> 在各种计算机体系结构中，对于字节、字等的存储机制有所不同，因而引发了计算机通信领域中一个很重要的问题，即通信双方交流的信息单元（比特、字节、字、双字等等）应该以什么样的顺序进行传送。如果不达成一致的规则，通信双方将无法进行正确的编/译码从而导致通信失败。
>
> -- 百度百科

不同的CPU有不同的字节序类型，这些字节序是指整数在内存中保存的顺序。

最常见的有两种

* Little-endian：将低序字节存储在起始地址（低位编址）
* Big-endian：将高序字节存储在起始地址（高位编址）

**LE（little-endian）：**

最符合人的思维的字节序：地址低位存储值的低位，地址高位存储值的高位。

怎么讲是最符合人的思维的字节序，是因为从人的第一观感来说：低位值小，就应该放在内存地址小的地方，也即内存地址低位；反之，高位值就应该放在内存地址大的地方，也即内存地址高位

**BE（big-endian）：**

最直观的字节序：地址低位存储值的高位，地址高位存储值的低位。

为什么说直观，不要考虑对应关系：只需要把内存地址从左到右按照由低到高的顺序写出，把值按照通常的高位到低位的顺序写出；两者对照，一个字节一个字节的填充进去

Unicode规范中定义，每一个文件的最前面分别加入一个表示编码顺序的字符，这个字符的名字叫做”零宽度非换行空格“（ZERO WIDTH NO-BREAK SPACE），用FEFF表示。这正好是两个字节，而且FF比FE大1。

如果一个文本文件的头两个字节是FE FF，就表示该文件采用大端模式；如果头两个字节是FF FE，就表示该文件采用小端模式。

### ASCII

> ASCII ((American Standard Code for Information Interchange): 美国信息交换标准代码）是基于拉丁字母的一套电脑编码系统，主要用于显示现代英语和其他西欧语言。它是最通用的信息交换标准，并等同于国际标准ISO/IEC 646。ASCII第一次以规范标准的类型发表是在1967年，最后一次更新则是在1986年，到目前为止共定义了128个字符 
>
>  -- 百度百科

### UTF-8

UTF-8是Unicode字符集的一种实现，它是一种变长字节编码。对于某一个字符的UTF-8编码，其编码规则为

* 如果只有一个字节则其最高二进制位为0
* 如果是多字节，其第一个字节从最高位开始，连续的二进制位值为1的个数决定了其编码的位数，其余各字节均以10开头。UTF-8最多可用到6个字节。

如：

1字节 0xxxxxxx 
2字节 110xxxxx 10xxxxxx 
3字节 1110xxxx 10xxxxxx 10xxxxxx 
4字节 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx 

所以，UTF-8是兼容ASCII的。

对于汉字来说，一个汉字需要3个code unit来表示，即对于UTF-8而言，一个汉字需要3byte的字节序列来编码

### UTF-16 & UTF-16be & UTF-16le

Java的运行时内部字符串就是用的UTF-16。即使JVM启动时，可以指定-Dfile.encoding=UTF-8或者别的，但这只是指定的读取文件或者输出字节序列，比如读取.class文件时的字符编码，在运行时，包括Java自己的序列化机制，就是UTF-16。

> UTF-16是Unicode字符编码五层次模型的第三层：字符编码表（Character Encoding Form，也称为 "storage format"）的一种实现方式。即把Unicode字符集的抽象码位映射为16位长的整数（即码元）的序列，用于数据存储或传递。Unicode字符的码位，需要1个或者2个16位长的码元来表示，因此这是一个变长表示。
>
> -- 百度百科

***对于汉字来说，一个汉字只需要一个UTF-16 code unit 即 2byte，所以，如果是需要保存一个很大的中文文档，使用UTF-16编码可以比使用UTF-8编码，节省一半的空间。***

UTF-16be:UTF-16的big endian编码模式

UTF-16le:UTF-16的little endian编码模式

### java中的相关方法

* String.length()：方法返回的是字符串中char的字符个数，也就是code unit的个数
* String.codePointCount()：方法返回字符串中的code point个数
* char String.charAt(index)：方法返回index位置的code unit
  int String.codePointAt(index)：方法返回index位置的code point。这也是这个方法的返回值是int而不是char的原因
* String.getBytes({charset})：返回字符串的字节序列长度。需要注意的是，此方法的无参重载默认使用jvm的file.encoding对应的字符集，一般是utf-8。所以，一个汉字，比如："莹".getBytes()，会返回三个字节；而"莹".getBytes("UTF-16")会返回4个字节，前两个字节是大小端标识。**对于一个字符串，只有第一个字符需要大小端标识 **；另外，"莹".getBytes("UTF-16be")或者是"莹".getBytes("UTF-16le")会返回两个字节，因为大小端已经制定，不需要额外再标识了。同理对于"中国人".getBytes("xxx")方法来说，utf-8/utf-16/utf-16be/utf-16le分别需要9/8/6/6个字节来表示
* new String(byte[],String):以制定的字符集来编码字节序列