### 打开一个数据库
```
#include <cassert>
#include "leveldb/db.h"

leveldb::DB* db;
leveldb::Options options;
options.create_if_missing = true;
leveldb::Status status = leveldb::DB::Open(options, "/tmp/testdb", &db);
assert(status.ok());  //用status检测操作是否成功
...
```

### 关闭数据库
```
... open the db as described above ...
... do something with db ...
delete db;
```
### 读和写
```
std::string value;
leveldb::Status s = db->Get(leveldb::ReadOptions(), key1, &value);//读取
if (s.ok()) s = db->Put(leveldb::WriteOptions(), key2, value);//写
if (s.ok()) s = db->Delete(leveldb::WriteOptions(), key1);//删除
```

### 原子更新？（Atomic Updates） 
没太看懂
```
#include "leveldb/write_batch.h"
...
std::string value;
leveldb::Status s = db->Get(leveldb::ReadOptions(), key1, &value);//保存待删除的键值的值
if (s.ok()) {
  leveldb::WriteBatch batch;
  batch.Delete(key1);//删除键值
  batch.Put(key2, value);//更改键值
  s = db->Write(leveldb::WriteOptions(), &batch);
}
```
WriteBatch存储着数据库待修改的操作

### 同步写
直到数据已经存储到硬盘才执行写操作
```
leveldb::WriteOptions write_options;
write_options.sync = true;
db->Put(write_options, ...);
```
异步写更快但是可能在异常时损失最后的数据
WriteBatch提供了异步写的另一种方式，多种更新放置在同一个WriteBatch上，然后使用同一个异步写，则可以平分花费

### 并发
待了解

### 迭代器显示所有数据
- 打印所有键-值对
```
leveldb::Iterator* it = db->NewIterator(leveldb::ReadOptions());
for (it->SeekToFirst(); it->Valid(); it->Next()) {
  cout << it->key().ToString() << ": "  << it->value().ToString() << endl;
}
assert(it->status().ok());  // Check for any errors found during the scan
delete it;
```
- 仅打印键
```
for (it->Seek(start);
   it->Valid() && it->key().ToString() < limit;
   it->Next()) {
  ...
}
```
- 逆向
```
for (it->SeekToLast(); it->Valid(); it->Prev()) {
  ...
}
```

### 快照
快照可以查看所有键值的储存情况
```
leveldb::ReadOptions options;
options.snapshot = db->GetSnapshot();
... apply some updates to db ...
leveldb::Iterator* iter = db->NewIterator(options); //使用iter来查看快照
... read using iter to view the state when the snapshot was created ...
delete iter;
db->ReleaseSnapshot(options.snapshot); //不需要快照时应释放
```

### 切片（slice）
切片是包含长度和指向外部字节数组的指针的数据结构，比string便宜
slice可和string和谐转换
```
leveldb::Slice s1 = "hello"; //String -> Slice

std::string str("world");
leveldb::Slice s2 = str;
```
```
std::string str = s1.ToString(); //Slice -> String
assert(str == std::string("hello"));
```

### 比较
键的排序按照字典的顺序，也可自行定义比较方法
比如键值有两位数，由第一位来排序，建立类来表达这些原则
```
class TwoPartComparator : public leveldb::Comparator {
 public:
  // Three-way comparison function:
  //   if a < b: negative result
  //   if a > b: positive result
  //   else: zero result
  int Compare(const leveldb::Slice& a, const leveldb::Slice& b) const {
    int a1, a2, b1, b2;
    ParseKey(a, &a1, &a2);
    ParseKey(b, &b1, &b2);
    if (a1 < b1) return -1;
    if (a1 > b1) return +1;
    if (a2 < b2) return -1;
    if (a2 > b2) return +1;
    return 0;
  }

  // Ignore the following methods for now:
  const char* Name() const { return "TwoPartComparator"; }
  void FindShortestSeparator(std::string*, const leveldb::Slice&) const {}
  void FindShortSuccessor(std::string*) const {}
};
```
接着使用自定义比较来创建一个数据库：
```
TwoPartComparator cmp;
leveldb::DB* db;
leveldb::Options options;
options.create_if_missing = true;
options.comparator = &cmp;
leveldb::Status status = leveldb::DB::Open(options, "/tmp/testdb", &db);
...
```
比较器可向下兼容？

### 性能
相邻的键聚集在同一块里，写硬盘前每块会独立地压缩
如果不需要压缩：
```
leveldb::Options options;
options.compression = leveldb::kNoCompression;
... leveldb::DB::Open(options, name, ...) ....
```
如果options.cache是非NULL,就会使用缓存来存储常使用的非压缩块内容
```
#include "leveldb/cache.h"

leveldb::Options options;
options.cache = leveldb::NewLRUCache(100 * 1048576);  // 100MB cache
leveldb::DB* db;
leveldb::DB::Open(options, name, &db);
... use the db ...
delete db
delete options.cache;
```
当需要大量读取时，应取消缓存
```
leveldb::ReadOptions options;
options.fill_cache = false;
leveldb::Iterator* it = db->NewIterator(options);
for (it->SeekToFirst(); it->Valid(); it->Next()) {
  ...
}
```



