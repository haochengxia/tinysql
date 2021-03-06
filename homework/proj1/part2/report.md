# Proj 1 Relational Algebra

> Date: 2021/03/06

## Part 2 Table Codec

### 1.1 基础概念

**唯一索引（unique index）**，如果能确定某个数据列将只包含彼此各不相同的值，在为这个数据列创建索引的时候就应该用关键字 UNIQUE 把它定义为一个唯一索引，好处有：
   1. 保证后续数据重复情况不出现；
   2. 唯一索引，比普通索引更有助于提高查询的效率。

> 问：为什么说唯一索引比普通索引更有助于提高查询的效率？
> 答：由于索引是有序的，加上唯一的特性，那么只要在叶子节点中扫描到数据之后直接回表取出数据，而普通索引还要尝试去扫描索引的叶子节点的下一条数据是否也满足要求。


**非唯一索引（non-unique index）**，也叫普通索引，普通索引（由关键字 KEY 或 INDEX 定义的索引）的唯一任务是加快对数据的访问速度，允许被索引的数据列包含重复的值。

### 1.2 Homework

作业描述见[Link](https://github.com/tidb-incubator/tinysql/blob/course/courses/proj1-part2-README-zh_CN.md)

#### 1.2.1 分析

因为要实现的两个函数为对两个编码函数结果的解码，因此先观察编码函数的行为：

```go
func EncodeRowKeyWithHandle(tableID int64, handle int64) kv.Key {
	buf := make([]byte, 0, RecordRowKeyLen)
	buf = appendTableRecordPrefix(buf, tableID)
	buf = codec.EncodeInt(buf, handle)
	return buf
}
```

上方为行数据编码函数，创建长度为19字节的buffer，其结构为：

| tablePrefix | tableID | recordPrefixSep| handle | 
|--|--|--|--|
|1 byte ("t")|8 bytes| 2 bytes ("_r") | 8 bytes |


```go
func EncodeIndexSeekKey(tableID int64, idxID int64, encodedValue []byte) kv.Key {
	key := make([]byte, 0, prefixLen+idLen+len(encodedValue))
	key = appendTableIndexPrefix(key, tableID)
	key = codec.EncodeInt(key, idxID)
	key = append(key, encodedValue...)
	return key
}
```

上方为索引数据编码函数，创建长度为19字节的buffer，其结构为：

| tablePrefix | tableID | indexPrefixSep| indexID | indexValues | 
|--|--|--|--|--|
|1 byte ("t")|8 bytes| 2 bytes ("_i") | 8 bytes | n byte(s) |

#### 1.2.2 实现

根据上述分析完成代码。需要注意`DecodeRecordKey`中判断`key`长度是否合法。

```go
func DecodeRecordKey(key kv.Key) (tableID int64, handle int64, err error) {
	/* My code start */
	if len(key) != RecordRowKeyLen {
		return 0, 0, errors.New("illegal key length")
	}

	buf := key
	_, handle, err = codec.DecodeInt(buf[len(buf)-8 : len(buf)])
	if err != nil {
		return 0, 0, errors.Trace(err)
	}

	buf = buf[:len(buf)-8]
	_, tableID, err = codec.DecodeInt(buf[tablePrefixLength:])
	if err != nil {
		return 0, 0, errors.Trace(err)
	}
	/* My code end */
	return
}
```

```go
func DecodeIndexKeyPrefix(key kv.Key) (tableID int64, indexID int64, indexValues []byte, err error) {
	/* My code start */
	indexValues = key[prefixLen+idLen:]

	buf := key[:prefixLen+idLen]
	_, indexID, err = codec.DecodeInt(buf[len(buf)-8 : len(buf)])
	if err != nil {
		return 0, 0, nil, errors.Trace(err)
	}

	buf = buf[:len(buf)-8]
	_, tableID, err = codec.DecodeInt(buf[tablePrefixLength:])
	if err != nil {
		return 0, 0, nil, errors.Trace(err)
	}
	/* My code end */
	return tableID, indexID, indexValues, nil
}
```

#### 1.2.3 结果

```bash
$ go test
OK: 12 passed
PASS
ok      github.com/pingcap/tidb/tablecodec      0.133s
```




