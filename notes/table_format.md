```
<beginning_of_file>
[data block 1]
[data block 2]
...
[data block N]
[meta block 1]
...
[meta block K]
[metaindex block]
[index block]
[Footer]        (fixed size; starts at file_size - sizeof(Footer))
<end_of_file>
```
每个文件包含内部指针。每个指针被称为块拔(BlockHandle)，并且包含下列信息：
```
offset:   varint64
size:     varint64
```
1. 键值有序排列并分成一系列的数据块，每个数据块按照`block_builder.cc`格式形成，并选择性压缩。
2. 数据块之后存着元块，格式同上
3. 元索引块包含着每个元块的入口，键是元块的名字，值是指向元块的块拔。
4. 索引块的每个入口包含着一个数据块，入口的键大于数据块的最后一个键，小于连着的数据块，值是块拔
5. 文件最后是定长的脚（footer),包含元索引和索引块的块拔和一个魔法数。
```
metaindex_handle: char[p];     // Block handle for metaindex
 index_handle:     char[q];     // Block handle for index
 padding:          char[40-p-q];// zeroed bytes to make fixed length
                                // (40==2*BlockHandle::kMaxEncodedLength)
 magic:            fixed64;     // == 0xdb4775248b80fb57 (little-endian)
 ```
 
 ### 过滤器，元块
 如果启用`FilterPolicy`，则每个表会有一个过滤块。元索引块包含着从`filter.<N>`到过滤块的块拔的映射入口，其中N为过滤准则`Name()`函数返回的数组。过滤块存储着一系列的过滤器。滤波器i包含着`FilterPolicy::CreateFilter()`在所有键的输出，输出存储在一个块里，块的偏置在
 `[ i*base ... (i+1)*base-1 ]`中
 如果base是2KB，则过滤块为
 ```
 [filter 0]
[filter 1]
[filter 2]
...
[filter N-1]

[offset of filter 0]                  : 4 bytes
[offset of filter 1]                  : 4 bytes
[offset of filter 2]                  : 4 bytes
...
[offset of filter N-1]                : 4 bytes

[offset of beginning of offset array] : 4 bytes
lg(base)                              : 1 byte
```
