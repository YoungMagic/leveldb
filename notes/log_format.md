log文件是一系列32KB的块，底部可能包含一个部分块。
每块包含着一系列的记录(record)：
```
block := record* trailer?
record :=
  checksum: uint32     // crc32c of type and data[] ; little-endian
  length: uint16       // little-endian低地址放低位
  type: uint8          // One of FULL, FIRST, MIDDLE, LAST
  data: uint8[length]
 ```
 记录从来不从一个块的最后六个字节内开始。任何遗留的字节应以0填充并被读者跳过。
 如果恰好只剩下7字节，并且添加了一个长度非0的记录，写需要发出一个FIRST记录（不包含数据）来填满这七个字，并且在之后的块里发出所有数据。
 更多的类型在之后加入
 ```
 FULL == 1
FIRST == 2
MIDDLE == 3
LAST == 4
```
FULL记录包含了整个用户记录的内容。FIRST, MIDDLE, LAST是分成多种小块的用户记录类型，FIRST是用户记录的第一个小块，LAST是最后，MIDDLE是用户记录的中间。

 
 

