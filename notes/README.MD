### 打开一个数据库
'''
#include <cassert>
#include "leveldb/db.h"

leveldb::DB* db;
leveldb::Options options;
options.create_if_missing = true;
leveldb::Status status = leveldb::DB::Open(options, "/tmp/testdb", &db);
assert(status.ok());  //用status检测操作是否成功
...
'''

### 关闭数据库
'''
... open the db as described above ...
... do something with db ...
delete db;
'''
### 读和写
'''
std::string value;
leveldb::Status s = db->Get(leveldb::ReadOptions(), key1, &value);//读取
if (s.ok()) s = db->Put(leveldb::WriteOptions(), key2, value);//写
if (s.ok()) s = db->Delete(leveldb::WriteOptions(), key1);//删除
'''

### 原子更新？（Atomic Updates） 
没太看懂
'''
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
'''
WriteBatch存储着数据库待修改的操作

### 同步写
直到数据已经存储到硬盘才执行写操作
'''
leveldb::WriteOptions write_options;
write_options.sync = true;
db->Put(write_options, ...);
'''
异步写更快但是可能在异常时损失最后的数据
WriteBatch提供了异步写的另一种方式，多种更新放置在同一个WriteBatch上，然后使用同一个异步写，则可以平分花费

### 并发
待了解

### 迭代器显示所有数据
- 打印所有键-值对
'''
leveldb::Iterator* it = db->NewIterator(leveldb::ReadOptions());
for (it->SeekToFirst(); it->Valid(); it->Next()) {
  cout << it->key().ToString() << ": "  << it->value().ToString() << endl;
}
assert(it->status().ok());  // Check for any errors found during the scan
delete it;
'''
- 仅打印键
'''
for (it->Seek(start);
   it->Valid() && it->key().ToString() < limit;
   it->Next()) {
  ...
}
'''
- 逆向
'''
for (it->SeekToLast(); it->Valid(); it->Prev()) {
  ...
}
'''




