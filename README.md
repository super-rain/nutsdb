# NutsDB
NutsDB is a simple, fast, embeddable and persistent key/value store
written in pure Go. It supports fully serializable transactions. All operations happen inside a Tx. Tx represents a transaction, which can be read-only or read-write. Read-only transactions can read values for a
given bucket and given key , or iterate over keys. Read-write transactions can update and delete keys from the DB.
It also supports range or prefix queries and TTL.

## Motivation
I wanted a simple, fast, embeddable and persistent key/value store written in pure Go. There are some options: 

BoltDB,it is based on B+ tree, has a good random read performance and awesome sequential scan performance, and it supports  ACID transactions with serializable isolation, but it is terrible at random write performance. 

GoLevelDB is based on a log-structured merge-tree (LSM tree), but it not supports transactions.

Badger is based on LSM tree with value log. It designed for SSDs. It also supports transactions. But in my [benchmark](https://github.com/xujiajun/nutsdb#benchmarks) its write performance is not as good as i thought. 

So i tried to build a kv store by myself, i wanted to find a simple store engine model as reference. 
Finally i found the bitcask model. It is very simple and easy to implement. Howerver it has its limition,like range or prefix queries are not effcient. For example, you can not easily scan over all keys between user000000 and user999999, you had to look up each key individully in the hashmap. 

In order to break the limition, i tried to optimize them. I tried to use B+ tree replace of hashmap and use mmap to optimize write performance. Finally i did it and named `NutsDB`. NutsDB offers a high read/write performance and supports ACID transactions. And it still has a lot of room for optimization. Welcome [contributions to NutsDB](https://github.com/xujiajun/nutsdb#contributing).

## Table of Contents

- [Getting Started](#getting-started)
  - [Installing](#installing)
  - [Opening a database](#opening-a-database)
  - [Transactions](#transactions)
    - [Read-write transactions](#read-write-transactions)
    - [Read-only transactions](#read-only-transactions)
    - [Managing transactions manually](#managing-transactions-manually)
  - [Using buckets](#using-buckets)
  - [Using key/value pairs](#using-keyvalue-pairs)
  - [Using TTL(Time To Live)](#using-ttltime-to-live)
  - [Iterating over keys](#iterating-over-keys)
    - [Prefix scans](#prefix-scans)
    - [Range scans](#range-scans)
  - [Merge Operation](#merge-operation)
  - [Database backup](#database-backup)
- [Using Other data structures](#using-other-data-structures)
   - [List](#list)
     - [RPush](#rpush)
     - [LPush](#lpush)
     - [LPop](#lpop)
     - [LPeek](#lpeek)
     - [RPop](#rpop)
     - [RPeek](#rpeek)
     - [LRange](#lrange)
     - [LRem](#lrem)
     - [LSet](#lset)	
     - [Ltrim](#ltrim)
     - [LSize](#lsize)  	
   - [Set](#set)
     - [SAdd](#sadd)
     - [SAreMembers](#saremembers)
     - [SCard](#scard)
     - [SDiffByOneBucket](#sdiffbyonebucket)
     - [SDiffByTwoBuckets](#sdiffbytwobuckets)
     - [SHasKey](#shaskey)
     - [SIsMember](#sismember)
     - [SMembers](#smembers)
     - [SMoveByOneBucket](#smovebyonebucket)
     - [SMoveByTwoBuckets](#smovebytwobuckets)
     - [SPop](#spop)
     - [SRem](#srem)
     - [SUnionByOneBucket](#sunionbyonebucket)
     - [SUnionByTwoBucket](#sunionbytwobuckets)
   - [Sorted Set](#sorted-set)
     - [ZAdd](#zadd)
     - [ZCard](#zcard)
     - [ZCount](#zcount)
     - [ZGetByKey](#zgetbykey)
     - [ZMembers](#zmembers)
     - [ZPeekMax](#zpeekmax)
     - [ZPeekMin](#zpeekmin)
     - [ZPopMax](#zpopmax)
     - [ZPopMin](#zpopmin)
     - [ZRangeByRank](#zrangebyrank)
     - [ZRangeByScore](#zrangebyscore)
     - [ZRank](#zrank)
     - [ZRem](#zrem)
     - [ZRemRangeByRank](#zremrangebyrank)
     - [ZScore](#zscore)
- [Comparison with other databases](#comparison-with-other-databases)
   - [BoltDB](#boltdb)
   - [LevelDB, RocksDB](#leveldb-rocksdb)
   - [Badger](#badger)
- [Benchmarks](#benchmarks)
- [Caveats & Limitations](#caveats--limitations)
- [Contact](#contact)
- [Contributing](#contributing)
- [Acknowledgements](#acknowledgements)
- [License](#license)

## Getting Started

### Installing

To start using NutsDB, first needs [Go](https://golang.org/dl/) installed (version 1.11+ is required).  and run go get:

```
go get -u github.com/xujiajun/nutsdb
```

### Opening a database

To open your database, use the nutsdb.Open() function,with the appropriate options.The `Dir` , `EntryIdxMode`  and  `SegmentSize`  options are must be specified by the client.

```golang
package main

import (
	"log"

	"github.com/xujiajun/nutsdb"
)

func main() {
	// Open the database located in the /tmp/nutsdb directory.
	// It will be created if it doesn't exist.
	opt := nutsdb.DefaultOptions
	opt.Dir = "/tmp/nutsdb"
	db, err := nutsdb.Open(opt)
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()

	...
}
```

### Transactions

NutsDB allows only one read-write transaction at a time but allows as many read-only transactions as you want at a time. Each transaction has a consistent view of the data as it existed when the transaction started.

When a transaction fails, it will roll back, and revert all changes that occurred to the database during that transaction.
When a read/write transaction succeeds all changes are persisted to disk.

Creating transaction from the `DB` is thread safe.

#### Read-write transactions

```golang
err := db.Update(
	func(tx *nutsdb.Tx) error {
	...
	return nil
})

```

#### Read-only transactions

```golang
err := db.View(
	func(tx *nutsdb.Tx) error {
	...
	return nil
})

```

#### Managing transactions manually

The `DB.View()`  and  `DB.Update()`  functions are wrappers around the  `DB.Begin()`  function. These helper functions will start the transaction, execute a function, and then safely close your transaction if an error is returned. This is the recommended way to use NutsDB transactions.

However, sometimes you may want to manually start and end your transactions. You can use the DB.Begin() function directly but please be sure to close the transaction. 

```golang
 // Start a write transaction.
tx, err := db.Begin(true)
if err != nil {
    return err
}

bucket := "bucket1"
key := []byte("foo")
val := []byte("bar")

// Use the transaction.
if err = tx.Put(bucket, key, val, Persistent); err != nil {
	// Rollback the transaction.
	tx.Rollback()
} else {
	// Commit the transaction and check for error.
	if err = tx.Commit(); err != nil {
		tx.Rollback()
		return err
	}
}
```

### Using buckets

Buckets are collections of key/value pairs within the database. All keys in a bucket must be unique.
Bucket can be interpreted as a table or namespace. So you can store the same key in different bucket. 

```golang

key := []byte("key001")
val := []byte("val001")

bucket001 := "bucket001"
if err := db.Update(
	func(tx *nutsdb.Tx) error {
		if err := tx.Put(bucket001, key, val, 0); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

bucket002 := "bucket002"
if err := db.Update(
	func(tx *nutsdb.Tx) error {
		if err := tx.Put(bucket002, key, val, 0); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

```

### Using key/value pairs

To save a key/value pair to a bucket, use the `tx.Put` method:

```golang

if err := db.Update(
	func(tx *nutsdb.Tx) error {
	key := []byte("name1")
	val := []byte("val1")
	bucket: = "bucket1"
	if err := tx.Put(bucket, key, val, 0); err != nil {
		return err
	}
	return nil
}); err != nil {
	log.Fatal(err)
}

```

This will set the value of the "name1" key to "val1" in the bucket1 bucket.

To update the the value of the "name1" key,we can still use the `tx.Put` function:

```golang
if err := db.Update(
	func(tx *nutsdb.Tx) error {
	key := []byte("name1")
	val := []byte("val1-modify") // Update the value
	bucket: = "bucket1"
	if err := tx.Put(bucket, key, val, 0); err != nil {
		return err
	}
	return nil
}); err != nil {
	log.Fatal(err)
}

```

To retrieve this value, we can use the `tx.Get` function:

```golang
if err := db.View(
func(tx *nutsdb.Tx) error {
	key := []byte("name1")
	bucket: = "bucket1"
	if e, err := tx.Get(bucket, key); err != nil {
		return err
	} else {
		fmt.Println(string(e.Value)) // "val1-modify"
	}
	return nil
}); err != nil {
	log.Println(err)
}
```

Use the `tx.Delete()` function to delete a key from the bucket.

```golang
if err := db.Update(
	func(tx *nutsdb.Tx) error {
	key := []byte("name1")
	bucket: = "bucket1"
	if err := tx.Delete(bucket, key); err != nil {
		return err
	}
	return nil
}); err != nil {
	log.Fatal(err)
}
```

### Using TTL(Time To Live)

NusDB supports TTL(Time to Live) for keys, you can use `tx.Put` function with a `ttl` parameter.

```golang
if err := db.Update(
	func(tx *nutsdb.Tx) error {
	key := []byte("name1")
	val := []byte("val1")
	bucket: = "bucket1"
	
	// If set ttl = 0 or Persistent, this key will nerver expired.
	// Set ttl = 60 , after 60 seconds, this key will expired.
	if err := tx.Put(bucket, key, val, 60); err != nil {
		return err
	}
	return nil
}); err != nil {
	log.Fatal(err)
}
```
### Iterating over keys

NutsDB stores its keys in byte-sorted order within a bucket. This makes sequential iteration over these keys extremely fast.

#### Prefix scans

To iterate over a key prefix, we can use `PrefixScan` function, and the paramter `limitNum` constrain the number of entries returned :

```golang

if err := db.View(
	func(tx *nutsdb.Tx) error {
		prefix := []byte("user_")
		// Constrain 100 entries returned 
		if entries, err := tx.PrefixScan(bucket, prefix, 100); err != nil {
			return err
		} else {
			keys, es := nutsdb.SortedEntryKeys(entries)
			for _, key := range keys {
				fmt.Println(key, string(es[key].Value))
			}
		}
		return nil
	}); err != nil {
		log.Fatal(err)
}

```

#### Range scans

To scan over a range, we can use `RangeScan` function. For example：

```golang
if err := db.View(
	func(tx *nutsdb.Tx) error {
		// Assume key from user_0000000 to user_9999999.
		// Query a specific user key range like this.
		start := []byte("user_0010001")
		end := []byte("user_0010010")
		bucket：= []byte("user_list)
		if entries, err := tx.RangeScan(bucket, start, end); err != nil {
			return err
		} else {
			keys, es := nutsdb.SortedEntryKeys(entries)
			for _, key := range keys {
				fmt.Println(key, string(es[key].Value))
			}
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```

### Merge Operation

NutsDB supports merge operation. you can use `db.Merge()` function removes dirty data and reduce data redundancy. Call this function from a read-write transaction. It will effect other write request. So you can execute it at the appropriate time.

```golang
err := db.Merge()
if err != nil {
    ...
}
```

### Database backup

NutsDB is easy to backup. You can use the `db.Backup()` function at given dir, call this function from a read-only transaction, it will perform a hot backup and not block your other database reads and writes.

```golang
err = db.Backup(dir)
if err != nil {
   ...
}
```

### Using other data structures

#### List

##### RPush

Inserts the values at the tail of the list stored in the bucket at given bucket,key and values.

```golang
if err := db.Update(
	func(tx *nutsdb.Tx) error {
		bucket := "bucketForList"
		key := []byte("myList")
		val := []byte("val1")
		if err := tx.RPush(bucket, key, val); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```

##### LPush 

Inserts the values at the head of the list stored in the bucket at given bucket,key and values.

```golang
if err := db.Update(
	func(tx *nutsdb.Tx) error {
	        bucket := "bucketForList"
		key := []byte("myList")
		val := []byte("val2")
		if err := tx.LPush(bucket, key, val); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```

##### LPop 

Removes and returns the first element of the list stored in the bucket at given bucket and key.

```golang
if err := db.Update(
	func(tx *nutsdb.Tx) error {
	        bucket := "bucketForList"
		key := []byte("myList")
		if item, err := tx.LPop(bucket, key); err != nil {
			return err
		} else {
			fmt.Println("LPop item:", string(item))
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```

##### LPeek

Returns the first element of the list stored in the bucket at given bucket and key.

```golang
if err := db.View(
	func(tx *nutsdb.Tx) error {
	        bucket := "bucketForList"
		key := []byte("myList")
		if item, err := tx.LPeek(bucket, key); err != nil {
			return err
		} else {
			fmt.Println("LPeek item:", string(item)) //val11
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```

##### RPop 

Removes and returns the last element of the list stored in the bucket at given bucket and key.

```golang
if err := db.Update(
	func(tx *nutsdb.Tx) error {
	        bucket := "bucketForList"
		key := []byte("myList")
		if item, err := tx.RPop(bucket, key); err != nil {
			return err
		} else {
			fmt.Println("RPop item:", string(item))
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```

##### RPeek

Returns the last element of the list stored in the bucket at given bucket and key.

```golang
if err := db.View(
	func(tx *nutsdb.Tx) error {
	        bucket := "bucketForList"
		key := []byte("myList")
		if item, err := tx.RPeek(bucket, key); err != nil {
			return err
		} else {
			fmt.Println("RPeek item:", string(item))
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```

##### LRange 

Returns the specified elements of the list stored in the bucket at given bucket,key, start and end.
The offsets start and stop are zero-based indexes 0 being the first element of the list (the head of the list),
1 being the next element and so on. 
Start and end can also be negative numbers indicating offsets from the end of the list,
where -1 is the last element of the list, -2 the penultimate element and so on.

```golang
if err := db.View(
	func(tx *nutsdb.Tx) error {
	        bucket := "bucketForList"
		key := []byte("myList")
		if items, err := tx.LRange(bucket, key, 0, -1); err != nil {
			return err
		} else {
			//fmt.Println(items)
			for _, item := range items {
				fmt.Println(string(item))
			}
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```
##### LRem 

Removes the first count occurrences of elements equal to value from the list stored in the bucket at given bucket,key,count.
The count argument influences the operation in the following ways:

* count > 0: Remove elements equal to value moving from head to tail.
* count < 0: Remove elements equal to value moving from tail to head.
* count = 0: Remove all elements equal to value.

```golang
if err := db.Update(
	func(tx *nutsdb.Tx) error {
	        bucket := "bucketForList"
		key := []byte("myList")
		if err := tx.LRem(bucket, key, 1); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```

##### LSet 

Sets the list element at index to value.

```golang
if err := db.Update(
	func(tx *nutsdb.Tx) error {
	        bucket := "bucketForList"
		key := []byte("myList")
		if err := tx.LSet(bucket, key, 0, []byte("val11")); err != nil {
			return err
		} else {
			fmt.Println("LSet ok, index 0 item value => val11")
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```

##### Ltrim 

Trims an existing list so that it will contain only the specified range of elements specified.
the offsets start and stop are zero-based indexes 0 being the first element of the list (the head of the list),
1 being the next element and so on.Start and end can also be negative numbers indicating offsets from the end of the list,
where -1 is the last element of the list, -2 the penultimate element and so on.

```golang
if err := db.Update(
	func(tx *nutsdb.Tx) error {
	        bucket := "bucketForList"
		key := []byte("myList")
		if err := tx.LTrim(bucket, key, 0, 1); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```

##### LSize 

Returns the size of key in the bucket in the bucket at given bucket and key.

```golang
if err := db.Update(
	func(tx *nutsdb.Tx) error {
	        bucket := "bucketForList"
		key := []byte("myList")
		if size,err := tx.LSize(bucket, key); err != nil {
			return err
		} else {
			fmt.Println("myList size is ",size)
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```

#### Set

##### SAdd

Adds the specified members to the set stored int the bucket at given bucket,key and items.

```go
if err := db.Update(
	func(tx *nutsdb.Tx) error {
	        bucket := "bucketForSet"
		key := []byte("mySet")
		if err := tx.SAdd(bucket, key, []byte("a"), []byte("b"), []byte("c")); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```

##### SAreMembers 

Returns if the specified members are the member of the set int the bucket at given bucket,key and items.

```go
if err := db.View(
	func(tx *nutsdb.Tx) error {
		bucket := "bucketForSet"
		key := []byte("mySet")
		if ok, err := tx.SAreMembers(bucket, key, []byte("a"), []byte("b"), []byte("c")); err != nil {
			return err
		} else {
			fmt.Println("SAreMembers:", ok)
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```

##### SCard 

```go

Returns the set cardinality (number of elements) of the set stored in the bucket at given bucket and key.

if err := db.View(
	func(tx *nutsdb.Tx) error {
		bucket := "bucketForSet"
		key := []byte("mySet")
		if num, err := tx.SCard(bucket, key); err != nil {
			return err
		} else {
			fmt.Println("SCard:", num)
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

```
##### SDiffByOneBucket 

Returns the members of the set resulting from the difference between the first set and all the successive sets in one bucket.

```go

key1 := []byte("mySet1")
key2 := []byte("mySet2")
bucket := "bucketForSet"

if err := db.Update(
	func(tx *nutsdb.Tx) error {
		if err := tx.SAdd(bucket, key1, []byte("a"), []byte("b"), []byte("c")); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

if err := db.Update(
	func(tx *nutsdb.Tx) error {
		if err := tx.SAdd(bucket, key2, []byte("c"), []byte("d")); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

if err := db.View(
	func(tx *nutsdb.Tx) error {
		if items, err := tx.SDiffByOneBucket(bucket, key1, key2); err != nil {
			return err
		} else {
			fmt.Println("SDiffByOneBucket:", items)
			for _, item := range items {
				fmt.Println("item", string(item))
			}
			//item a
			//item b
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

```

##### SDiffByTwoBuckets 

Returns the members of the set resulting from the difference between the first set and all the successive sets in two buckets.

```go
bucket1 := "bucket1"
key1 := []byte("mySet1")

bucket2 := "bucket2"
key2 := []byte("mySet2")

if err := db.Update(
	func(tx *nutsdb.Tx) error {
		if err := tx.SAdd(bucket1, key1, []byte("a"), []byte("b"), []byte("c")); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

if err := db.Update(
	func(tx *nutsdb.Tx) error {
		if err := tx.SAdd(bucket2, key2, []byte("c"), []byte("d")); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

if err := db.View(
	func(tx *nutsdb.Tx) error {
		if items, err := tx.SDiffByTwoBuckets(bucket1, key1, bucket2, key2); err != nil {
			return err
		} else {
			fmt.Println("SDiffByTwoBuckets:", items)
			for _, item := range items {
				fmt.Println("item", string(item))
			}
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

```
##### SHasKey 

Returns if the set in the bucket at given bucket and key.

```go

bucket := "bucketForSet"

if err := db.View(
	func(tx *nutsdb.Tx) error {
		if ok, err := tx.SHasKey(bucket, []byte("mySet")); err != nil {
			return err
		} else {
			fmt.Println("SHasKey", ok)
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

```
##### SIsMember 

Returns if member is a member of the set stored int the bucket at given bucket,key and item.

```go
bucket := "bucketForSet"

if err := db.View(
	func(tx *nutsdb.Tx) error {
		if ok, err := tx.SIsMember(bucket, []byte("mySet"), []byte("a")); err != nil {
			return err
		} else {
			fmt.Println("SIsMember", ok)
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```
##### SMembers 

Returns all the members of the set value stored int the bucket at given bucket and key.

```
bucket := "bucketForSet"

if err := db.View(
	func(tx *nutsdb.Tx) error {
		if items, err := tx.SMembers(bucket, []byte("mySet")); err != nil {
			return err
		} else {
			fmt.Println("SMembers", items)
			for _, item := range items {
				fmt.Println("item", string(item))
			}
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```
##### SMoveByOneBucket 

Moves member from the set at source to the set at destination in one bucket.

```go
bucket3 := "bucket3"

if err := db.Update(
	func(tx *nutsdb.Tx) error {
		if err := tx.SAdd(bucket3, []byte("mySet1"), []byte("a"), []byte("b"), []byte("c")); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
if err := db.Update(
	func(tx *nutsdb.Tx) error {
		if err := tx.SAdd(bucket3, []byte("mySet2"), []byte("c"), []byte("d"), []byte("e")); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

if err := db.Update(
	func(tx *nutsdb.Tx) error {
		if ok, err := tx.SMoveByOneBucket(bucket3, []byte("mySet1"), []byte("mySet2"), []byte("a")); err != nil {
			return err
		} else {
			fmt.Println("SMoveByOneBucket", ok)
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

if err := db.View(
	func(tx *nutsdb.Tx) error {
		if items, err := tx.SMembers(bucket3, []byte("mySet1")); err != nil {
			return err
		} else {
			fmt.Println("after SMoveByOneBucket bucket3 mySet1 SMembers", items)
			for _, item := range items {
				fmt.Println("item", string(item))
			}
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

if err := db.View(
	func(tx *nutsdb.Tx) error {
		if items, err := tx.SMembers(bucket3, []byte("mySet2")); err != nil {
			return err
		} else {
			fmt.Println("after SMoveByOneBucket bucket3 mySet2 SMembers", items)
			for _, item := range items {
				fmt.Println("item", string(item))
			}
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```
##### SMoveByTwoBuckets 

Moves member from the set at source to the set at destination in two buckets.

```go
bucket4 := "bucket4"
bucket5 := "bucket5"
if err := db.Update(
	func(tx *nutsdb.Tx) error {
		if err := tx.SAdd(bucket4, []byte("mySet1"), []byte("a"), []byte("b"), []byte("c")); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
if err := db.Update(
	func(tx *nutsdb.Tx) error {
		if err := tx.SAdd(bucket5, []byte("mySet2"), []byte("c"), []byte("d"), []byte("e")); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

if err := db.Update(
	func(tx *nutsdb.Tx) error {
		if ok, err := tx.SMoveByTwoBuckets(bucket4, []byte("mySet1"), bucket5, []byte("mySet2"), []byte("a")); err != nil {
			return err
		} else {
			fmt.Println("SMoveByTwoBuckets", ok)
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

if err := db.View(
	func(tx *nutsdb.Tx) error {
		if items, err := tx.SMembers(bucket4, []byte("mySet1")); err != nil {
			return err
		} else {
			fmt.Println("after SMoveByTwoBuckets bucket4 mySet1 SMembers", items)
			for _, item := range items {
				fmt.Println("item", string(item))
			}
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

if err := db.View(
	func(tx *nutsdb.Tx) error {
		if items, err := tx.SMembers(bucket5, []byte("mySet2")); err != nil {
			return err
		} else {
			fmt.Println("after SMoveByTwoBuckets bucket5 mySet2 SMembers", items)
			for _, item := range items {
				fmt.Println("item", string(item))
			}
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```
##### SPop 

Removes and returns one or more random elements from the set value store in the bucket at given bucket and key.

```go
if err := db.Update(
	func(tx *nutsdb.Tx) error {
		key := []byte("mySet")
		if item, err := tx.SPop(bucket, key); err != nil {
			return err
		} else {
			fmt.Println("SPop item from mySet:", string(item))
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```
##### SRem 

Removes the specified members from the set stored int the bucket at given bucket,key and items.

```go
bucket6:="bucket6"
if err := db.Update(
	func(tx *nutsdb.Tx) error {
		if err := tx.SAdd(bucket6, []byte("mySet"), []byte("a"), []byte("b"), []byte("c")); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

if err := db.Update(
	func(tx *nutsdb.Tx) error {
		if err := tx.SRem(bucket6, []byte("mySet"), []byte("a")); err != nil {
			return err
		} else {
			fmt.Println("SRem ok")
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

if err := db.View(
	func(tx *nutsdb.Tx) error {
		if items, err := tx.SMembers(bucket6, []byte("mySet")); err != nil {
			return err
		} else {
			fmt.Println("SMembers items:", items)
			for _, item := range items {
				fmt.Println("item:", string(item))
			}
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```
##### SUnionByOneBucket 

The members of the set resulting from the union of all the given sets in one bucket.

```go
bucket7 := "bucket1"
key1 := []byte("mySet1")
key2 := []byte("mySet2")

if err := db.Update(
	func(tx *nutsdb.Tx) error {
		if err := tx.SAdd(bucket7, key1, []byte("a"), []byte("b"), []byte("c")); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

if err := db.Update(
	func(tx *nutsdb.Tx) error {
		if err := tx.SAdd(bucket7, key2, []byte("c"), []byte("d")); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

if err := db.View(
	func(tx *nutsdb.Tx) error {
		if items, err := tx.SUnionByOneBucket(bucket7, key1, key2); err != nil {
			return err
		} else {
			fmt.Println("SUnionByOneBucket:", items)
			for _, item := range items {
				fmt.Println("item", string(item))
			}
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```

##### SUnionByTwoBuckets 

The members of the set resulting from the union of all the given sets in two buckets.

```go
bucket8 := "bucket1"
key1 := []byte("mySet1")

bucket9 := "bucket2"
key2 := []byte("mySet2")

if err := db.Update(
	func(tx *nutsdb.Tx) error {
		if err := tx.SAdd(bucket8, key1, []byte("a"), []byte("b"), []byte("c")); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

if err := db.Update(
	func(tx *nutsdb.Tx) error {
		if err := tx.SAdd(bucket9, key2, []byte("c"), []byte("d")); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

if err := db.View(
	func(tx *nutsdb.Tx) error {
		if items, err := tx.SUnionByTwoBuckets(bucket8, key1, bucket9, key2); err != nil {
			return err
		} else {
			fmt.Println("SUnionByTwoBucket:", items)
			for _, item := range items {
				fmt.Println("item", string(item))
			}
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```

#### Sorted Set

##### ZAdd

Adds the specified member with the specified score and the specified value to the sorted set stored at key.

```go
if err := db.Update(
	func(tx *nutsdb.Tx) error {
		bucket := "myZSet1"
		key := []byte("key1")
		if err := tx.ZAdd(bucket, key, 1, []byte("val1")); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```
##### ZCard 

Returns the sorted set cardinality (number of elements) of the sorted set stored at key.

```go
if err := db.View(
	func(tx *nutsdb.Tx) error {
		bucket := "myZSet1"
		if num, err := tx.ZCard(bucket); err != nil {
			return err
		} else {
			fmt.Println("ZCard num", num)
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```

##### ZCount 

Returns the number of elements in the sorted set at key with a score between min and max and opts.

Opts includes the following parameters:

* Limit        int  // limit the max nodes to return
* ExcludeStart bool // exclude start value, so it search in interval (start, end] or (start, end)
* ExcludeEnd   bool // exclude end value, so it search in interval [start, end) or (start, end)

```go
if err := db.View(
	func(tx *nutsdb.Tx) error {
		bucket := "myZSet1"
		if num, err := tx.ZCount(bucket, 0, 1, nil); err != nil {
			return err
		} else {
			fmt.Println("ZCount num", num)
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```
##### ZGetByKey 

Returns node in the bucket at given bucket and key.

```go
if err := db.View(
	func(tx *nutsdb.Tx) error {
		bucket := "myZSet1"
		key := []byte("key2")
		if node, err := tx.ZGetByKey(bucket, key); err != nil {
			return err
		} else {
			fmt.Println("ZGetByKey key2 val:", string(node.Value))
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```
##### ZMembers 

Returns all the members of the set value stored at key.

```go
if err := db.View(
	func(tx *nutsdb.Tx) error {
		bucket := "myZSet1"
		if nodes, err := tx.ZMembers(bucket); err != nil {
			return err
		} else {
			fmt.Println("ZMembers:", nodes)

			for _, node := range nodes {
				fmt.Println("member:", node.Key(), string(node.Value))
			}
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```
##### ZPeekMax 

Returns up to count members with the highest scores in the sorted set stored at key.

```go
if err := db.View(
	func(tx *nutsdb.Tx) error {
		bucket := "myZSet1"
		if node, err := tx.ZPeekMax(bucket); err != nil {
			return err
		} else {
			fmt.Println("ZPeekMax:", string(node.Value))
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```

##### ZPeekMin 

Returns up to count members with the lowest scores in the sorted set stored at key.

```go
if err := db.View(
	func(tx *nutsdb.Tx) error {
		bucket := "myZSet1"
		if node, err := tx.ZPeekMin(bucket); err != nil {
			return err
		} else {
			fmt.Println("ZPeekMin:", string(node.Value))
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```

##### ZPopMax 

Removes and returns up to count members with the highest scores in the sorted set stored at key.

```go
if err := db.Update(
	func(tx *nutsdb.Tx) error {
		bucket := "myZSet1"
		if node, err := tx.ZPopMax(bucket); err != nil {
			return err
		} else {
			fmt.Println("ZPopMax:", string(node.Value)) //val3
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```
##### ZPopMin 

Removes and returns up to count members with the lowest scores in the sorted set stored at key.

```go
if err := db.Update(
	func(tx *nutsdb.Tx) error {
		bucket := "myZSet1"
		if node, err := tx.ZPopMin(bucket); err != nil {
			return err
		} else {
			fmt.Println("ZPopMin:", string(node.Value)) //val1
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```

##### ZRangeByRank 

returns all the elements in the sorted set in one bucket at bucket and key with a rank between start and end (including elements with rank equal to start or end).

```go
// ZAdd add items
if err := db.Update(
	func(tx *nutsdb.Tx) error {
		bucket := "myZSet2"
		key1 := []byte("key1")
		if err := tx.ZAdd(bucket, key1, 1, []byte("val1")); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

if err := db.Update(
	func(tx *nutsdb.Tx) error {
		bucket := "myZSet2"
		key2 := []byte("key2")
		if err := tx.ZAdd(bucket, key2, 2, []byte("val2")); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

if err := db.Update(
	func(tx *nutsdb.Tx) error {
		bucket := "myZSet2"
		key3 := []byte("key3")
		if err := tx.ZAdd(bucket, key3, 3, []byte("val3")); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

// ZRangeByRank
if err := db.View(
	func(tx *nutsdb.Tx) error {
		bucket := "myZSet2"
		if nodes, err := tx.ZRangeByRank(bucket, 1, 2); err != nil {
			return err
		} else {
			fmt.Println("ZRangeByRank nodes :", nodes)
			for _, node := range nodes {
				fmt.Println("item:", node.Key(), node.Score())
			}
			
			//item: key1 1
			//item: key2 2
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```

##### ZRangeByScore 

Returns all the elements in the sorted set at key with a score between min and max.
And the parameter `Opts` is the same as ZCount's.

```go
// ZAdd
if err := db.Update(
		func(tx *nutsdb.Tx) error {
			bucket := "myZSet3"
			key1 := []byte("key1")
			if err := tx.ZAdd(bucket, key1, 70, []byte("val1")); err != nil {
				return err
			}
			return nil
		}); err != nil {
		log.Fatal(err)
	}

if err := db.Update(
	func(tx *nutsdb.Tx) error {
		bucket := "myZSet3"
		key2 := []byte("key2")
		if err := tx.ZAdd(bucket, key2, 90, []byte("val2")); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

if err := db.Update(
	func(tx *nutsdb.Tx) error {
		bucket := "myZSet3"
		key3 := []byte("key3")
		if err := tx.ZAdd(bucket, key3, 86, []byte("val3")); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

// ZRangeByScore
if err := db.View(
	func(tx *nutsdb.Tx) error {
		bucket := "myZSet3"
		if nodes, err := tx.ZRangeByScore(bucket, 80, 100,nil); err != nil {
			return err
		} else {
			fmt.Println("ZRangeByScore nodes :", nodes)
			for _, node := range nodes {
				fmt.Println("item:", node.Key(), node.Score())
			}
			//item: key3 86
			//item: key2 90
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}	
```
##### ZRank

Returns the rank of member in the sorted set stored in the bucket at given bucket and key.

```go
if err := db.View(
	func(tx *nutsdb.Tx) error {
		bucket := "myZSet4"
		key1 := []byte("key1")
		if rank, err := tx.ZRank(bucket, key1); err != nil {
			return err
		} else {
			fmt.Println("key1 ZRank :", rank)
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```
##### ZRem 

Removes the specified members from the sorted set stored in one bucket at given bucket and key.

```go
if err := db.Update(
	func(tx *nutsdb.Tx) error {
		bucket := "myZSet5"
		key1 := []byte("key1")
		if err := tx.ZAdd(bucket, key1, 10, []byte("val1")); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

if err := db.Update(
	func(tx *nutsdb.Tx) error {
		bucket := "myZSet5"
		key2 := []byte("key2")
		if err := tx.ZAdd(bucket, key2, 20, []byte("val2")); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

if err := db.View(
	func(tx *nutsdb.Tx) error {
		bucket := "myZSet5"
		if nodes,err := tx.ZMembers(bucket); err != nil {
			return err
		} else {
			fmt.Println("before ZRem key1, ZMembers nodes",nodes)
			for _,node:=range nodes {
				fmt.Println("item:",node.Key(),node.Score())
			}
		}
		// before ZRem key1, ZMembers nodes map[key1:0xc00008cfa0 key2:0xc00008d090]
		// item: key1 10
		// item: key2 20
		return nil
	}); err != nil {
	log.Fatal(err)
}

if err := db.Update(
	func(tx *nutsdb.Tx) error {
		bucket := "myZSet5"
		if err := tx.ZRem(bucket, "key1"); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

if err := db.View(
	func(tx *nutsdb.Tx) error {
		bucket := "myZSet5"
		if nodes,err := tx.ZMembers(bucket); err != nil {
			return err
		} else {
			fmt.Println("after ZRem key1, ZMembers nodes",nodes)
			for _,node:=range nodes {
				fmt.Println("item:",node.Key(),node.Score())
			}
			// after ZRem key1, ZMembers nodes map[key2:0xc00008d090]
			// item: key2 20
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

```

##### ZRemRangeByRank 

Removes all elements in the sorted set stored in one bucket at given bucket with rank between start and end.
The rank is 1-based integer. Rank 1 means the first node; Rank -1 means the last node.

```go
if err := db.Update(
	func(tx *nutsdb.Tx) error {
		bucket := "myZSet6"
		key1 := []byte("key1")
		if err := tx.ZAdd(bucket, key1, 10, []byte("val1")); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

if err := db.Update(
	func(tx *nutsdb.Tx) error {
		bucket := "myZSet6"
		key2 := []byte("key2")
		if err := tx.ZAdd(bucket, key2, 20, []byte("val2")); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

if err := db.Update(
	func(tx *nutsdb.Tx) error {
		bucket := "myZSet6"
		key3 := []byte("key3")
		if err := tx.ZAdd(bucket, key3, 30, []byte("val2")); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

if err := db.View(
	func(tx *nutsdb.Tx) error {
		bucket := "myZSet6"
		if nodes,err := tx.ZMembers(bucket); err != nil {
			return err
		} else {
			fmt.Println("before ZRemRangeByRank, ZMembers nodes",nodes)
			for _,node:=range nodes {
				fmt.Println("item:",node.Key(),node.Score())
			}
			// before ZRemRangeByRank, ZMembers nodes map[key3:0xc00008d450 key1:0xc00008d270 key2:0xc00008d360]
			// item: key1 10
			// item: key2 20
			// item: key3 30
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

if err := db.Update(
	func(tx *nutsdb.Tx) error {
		bucket := "myZSet6"
		if err := tx.ZRemRangeByRank(bucket, 1,2); err != nil {
			return err
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}

if err := db.View(
	func(tx *nutsdb.Tx) error {
		bucket := "myZSet6"
		if nodes,err := tx.ZMembers(bucket); err != nil {
			return err
		} else {
			fmt.Println("after ZRemRangeByRank, ZMembers nodes",nodes)
			for _,node:=range nodes {
				fmt.Println("item:",node.Key(),node.Score())
			}
			// after ZRemRangeByRank, ZMembers nodes map[key3:0xc00008d450]
			// item: key3 30
			// key1 ZScore 10
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```
##### ZScore

Returns the score of member in the sorted set in the bucket at given bucket and key.

```go
if err := db.View(
	func(tx *nutsdb.Tx) error {
		bucket := "myZSet7"
		if score,err := tx.ZScore(bucket, []byte("key1")); err != nil {
			return err
		} else {
			fmt.Println("ZScore key1 score:",score)
		}
		return nil
	}); err != nil {
	log.Fatal(err)
}
```
### Comparison with other databases

#### BoltDB

BoltDB is similar to NutsDB, both use B+tree and support transaction. However, Bolt uses a B+tree internally and only a single file, and NutsDB is based on bitcask model with  multiple log files. NutsDB supports TTL, but BoltDB not support it . NutsDB offers high-performance reads and writes, but BoltDb writes performance not so good.

#### LevelDB, RocksDB

LevelDB and RocksDB are based on a log-structured merge-tree (LSM tree).An LSM tree optimizes random writes by using a write ahead log and multi-tiered, sorted files called SSTables. LevelDB does not have transactions. It supports batch writing of key/values pairs and it supports read snapshots but it will not give you the ability to do a compare-and-swap operation safely. NutsDB supports fully serializable ACID transactions.

#### Badger

Badger is based in LSM tree with value log. It designed for SSDs. It also supports transaction and TTL. But in my benchmark its write performance is not as good as i thought.

### Benchmarks

#### Tested kvstore

* [BadgerDB](https://github.com/dgraph-io/badger) (with default options)
* [BoltDB](https://github.com/boltdb/bolt) (with default options)
* [BuntDB](https://github.com/tidwall/buntdb) (with default options)
* [LevelDB](https://github.com/syndtr/goleveldb) (with default options)
* [NutsDB](https://github.com/xujiajun/nutsdb) (with default options or custom options)

#### Benchmark System:

* Go Version : go1.11.4 darwin/amd64
* OS: Mac OS X 10.13.6
* Architecture: x86_64
* 16 GB 2133 MHz LPDDR3
* CPU: 3.1 GHz Intel Core i7

#### Benchmark results:

```
BenchmarkBadgerDBPutValue64B-8    	   10000	    135431 ns/op	    2375 B/op	      74 allocs/op
BenchmarkBadgerDBPutValue128B-8   	   10000	    119450 ns/op	    2503 B/op	      74 allocs/op
BenchmarkBadgerDBPutValue256B-8   	   10000	    142451 ns/op	    2759 B/op	      74 allocs/op
BenchmarkBadgerDBPutValue512B-8   	   10000	    109066 ns/op	    3270 B/op	      74 allocs/op
BenchmarkBadgerDBGet-8            	 1000000	      1679 ns/op	     416 B/op	       9 allocs/op
BenchmarkBoltDBPutValue64B-8      	    5000	    200487 ns/op	   20005 B/op	      59 allocs/op
BenchmarkBoltDBPutValue128B-8     	    5000	    230297 ns/op	   13703 B/op	      64 allocs/op
BenchmarkBoltDBPutValue256B-8     	    5000	    207220 ns/op	   16708 B/op	      64 allocs/op
BenchmarkBoltDBPutValue512B-8     	    5000	    262358 ns/op	   17768 B/op	      64 allocs/op
BenchmarkBoltDBGet-8              	 1000000	      1163 ns/op	     592 B/op	      10 allocs/op
BenchmarkBoltDBRangeScans-8       	 1000000	      1226 ns/op	     584 B/op	       9 allocs/op
BenchmarkBoltDBPrefixScans-8      	 1000000	      1275 ns/op	     584 B/op	       9 allocs/op
BenchmarkBuntDBPutValue64B-8      	  200000	      8930 ns/op	     927 B/op	      14 allocs/op
BenchmarkBuntDBPutValue128B-8     	  200000	      8892 ns/op	    1015 B/op	      15 allocs/op
BenchmarkBuntDBPutValue256B-8     	  200000	     11282 ns/op	    1274 B/op	      16 allocs/op
BenchmarkBuntDBPutValue512B-8     	  200000	     12323 ns/op	    1794 B/op	      16 allocs/op
BenchmarkBuntDBGet-8              	 2000000	       675 ns/op	     104 B/op	       4 allocs/op
BenchmarkLevelDBPutValue64B-8     	  100000	     11909 ns/op	     476 B/op	       7 allocs/op
BenchmarkLevelDBPutValue128B-8    	  200000	     10838 ns/op	     254 B/op	       7 allocs/op
BenchmarkLevelDBPutValue256B-8    	  100000	     11510 ns/op	     445 B/op	       7 allocs/op
BenchmarkLevelDBPutValue512B-8    	  100000	     12661 ns/op	     799 B/op	       8 allocs/op
BenchmarkLevelDBGet-8             	 1000000	      1371 ns/op	     184 B/op	       5 allocs/op
BenchmarkNutsDBPutValue64B-8      	 1000000	      2472 ns/op	     670 B/op	      14 allocs/op
BenchmarkNutsDBPutValue128B-8     	 1000000	      2182 ns/op	     664 B/op	      13 allocs/op
BenchmarkNutsDBPutValue256B-8     	 1000000	      2579 ns/op	     920 B/op	      13 allocs/op
BenchmarkNutsDBPutValue512B-8     	 1000000	      3640 ns/op	    1432 B/op	      13 allocs/op
BenchmarkNutsDBGet-8              	 2000000	       781 ns/op	      88 B/op	       3 allocs/op
BenchmarkNutsDBGetByMemoryMap-8   	   50000	     40734 ns/op	     888 B/op	      17 allocs/op
BenchmarkNutsDBPrefixScan-8       	 1000000	      1293 ns/op	     656 B/op	       9 allocs/op
BenchmarkNutsDBRangeScan-8        	 1000000	      2250 ns/op	     752 B/op	      12 allocs/op
```

#### Conclusions:

* Put(write) Performance:  NutsDB、BuntDB、LevelDB are fast. And NutsDB is fastest. It is 5-10x faster than LevelDB, 5x faster than BuntDB.

*  Get(read) Performance: All are fast. And NutsDB and BuntDB with default options are 2x faster than others.
And NutsDB reads with MemoryMap option is slower 40x than its default option way. 


The benchmarking code can be found in the [gokvstore-bench](https://github.com/xujiajun/gokvstore-bench) repo.

### Caveats & Limitations

NutsDB supports two modes about entry index: `HintAndRAMIdxMode`  and  `HintAndMemoryMapIdxMode`. The default mode use `HintAndRAMIdxMode`, entries are indexed base on RAM, so its read/write performance is fast. but can’t handle databases much larger than the available physical RAM. If you set the `HintAndMemoryMapIdxMode` mode, HintIndex will not cache the value of the entry. Its write performance is also fast. To retrieve a key by seeking to offset relative to the start of the data file, so its read performance more slowly that RAM way, but it can handle databases much larger than the available physical RAM.

NutsDB will truncate data file if the active file is larger than  `SegmentSize`, so the size of an entry can not be set larger than `SegmentSize` , defalut `SegmentSize` is 64MB, you can set it(opt.SegmentSize) as option before DB opening. Once set, it cannot be changed.

### Contact

* [xujiajun](https://github.com/xujiajun)

### Contributing

Welcomes contributions to NutsDB.

#### Issues
Feel free to submit [issues](https://github.com/xujiajun/nutsdb/issues) and enhancement requests.

#### Contribution flow

This is a rough outline of what a contributor's workflow looks like:

* 1.**Fork** the repo on GitHub
* 2.**Clone** the project to your own machine
* 3.**Commit** changes to your own branch
* 4.**Push** your work back up to your fork
* 5.Submit a **Pull request** so that we can review your changes

Thanks for contributing!

### Acknowledgements

This package is inspired by the following:

* [Bitcask-intro](https://github.com/basho/bitcask/blob/develop/doc/bitcask-intro.pdf)
* [BoltDB](https://github.com/boltdb)
* [BuntDB](https://github.com/tidwall/buntdb)

### License

The NutsDB is open-sourced software licensed under the [Apache 2.0 license](https://github.com/xujiajun/nutsdb/blob/master/LICENSE).
