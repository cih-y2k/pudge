[![Documentation](https://godoc.org/github.com/recoilme/pudge?status.svg)](https://godoc.org/github.com/recoilme/pudge)
[![Go Report Card](https://goreportcard.com/badge/github.com/recoilme/pudge)](https://goreportcard.com/report/github.com/recoilme/pudge)
[![Build Status](https://travis-ci.org/recoilme/pudge.svg?branch=master)](https://travis-ci.org/recoilme/pudge)

**Description**

Package pudge is a simple key/value store written using Go's standard library only. Keys are stored in memory (with persistence), values stored on disk.

It presents the following:
* Supporting very efficient lookup, insertions and deletions
* Performance is comparable to hash tables
* Ability to get the data in sorted order, which enables additional operations like range scan
* Select with limit/offset/from key, with ordering or by prefix
* Safe for use in goroutines
* Space efficient
* Very short and simple codebase
* Well tested, used in production

![pudge](https://avatars3.githubusercontent.com/u/417177?s=460&v=4)

**Usage**


```
package main

import (
	"log"

	"github.com/recoilme/pudge"
)

func main() {
	ExampleSet()
	ExampleGet()
	ExampleOpen()
}

//ExampleSet (lazy open)
func ExampleSet() {
	pudge.Set("../test/test", "Hello", "World")
	defer pudge.CloseAll()
}

//ExampleGet (lazy open)
func ExampleGet() {
	output := ""
	pudge.Get("../test/test", "Hello", &output)
	log.Println("Output:", output)
	// Output: World
	defer pudge.CloseAll()
}

//ExampleOpen (complex example)
func ExampleOpen() {
	cfg := pudge.DefaultConfig()
	cfg.SyncInterval = 0 //disable every second fsync
	db, err := pudge.Open("../test/db", cfg)
	if err != nil {
		log.Panic(err)
	}
	defer db.DeleteFile()
	type Point struct {
		X int
		Y int
	}
	for i := 100; i >= 0; i-- {
		p := &Point{X: i, Y: i}
		db.Set(i, p)
	}
	var point Point
	db.Get(8, &point)
	log.Println(point)
	// Output: {8 8}
	// Select 2 keys, from 7 in ascending order
	keys, _ := db.Keys(7, 2, 0, true)
	for _, key := range keys {
		var p Point
		db.Get(key, &p)
		log.Println(p)
	}
	// Output: {8 8}
	// Output: {9 9}
}

```


**Benchmarks**


[All tests here](https://github.com/recoilme/pogreb-bench)

Some tests, MacBook Pro (Retina, 13-inch, Early 2015)
=====================================================


### Test 1
Number of keys: 1000000
Minimum key size: 16, maximum key size: 64
Minimum value size: 128, maximum value size: 512
Concurrency: 2


|                       | pogreb  | goleveldb | bolt   | badgerdb | pudge  | slowpoke | pudge(mem) |
|-----------------------|---------|-----------|--------|----------|--------|----------|------------|
| 1M (Put+Get), seconds | 187     | 38        | 126    | 34       | 23     | 23       | 2          |
| 1M Put, ops/sec       | 5336    | 34743     | 8054   | 33539    | 47298  | 46789    | 439581     |
| 1M Get, ops/sec       | 1782423 | 98406     | 499871 | 220597   | 499172 | 445783   | 1652069    |
| FileSize,Mb           | 568     | 357       | 552    | 487      | 358    | 358      | 358        |


### Test 4
Number of keys: 10000000
Key size: 8
Value size: 16
Concurrency: 100


|                       | goleveldb | badgerdb | pudge  |
|-----------------------|-----------|----------|--------|
| 10M (Put+Get), seconds| 165       | 120      | 243    |
| 10M Put, ops/sec      | 122933    | 135709   | 43843  |
| 10M Get, ops/sec      | 118722    | 214981   | 666067 |
| FileSize,Mb           | 312       | 1370     | 381    |