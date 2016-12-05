---
layout: post
title:  "Go Concurrency: Buffering Streams by Time and Count"
date:   2016-12-01 12:00:00
categories: etl pipelines golang data
---

The Go Blog has released some great articles on concurrency patterns; thanks Go Team! The post on streaming [pipelines](https://blog.golang.org/pipelines)
illustrates the simplicity of processing data across different stages in the application. 

In the examples, stages are encapsulated by functions that ingest and create channels:
```
// producer 
func gen(cfg Config) <-chan T1  { ... }

// flow stage 
func proc(c <-chan T1) <-chan T2  { ... }

// sink 
func sink(c <-chan T2) {
	for t := range c {
		fmt.Println(<-c)
	}
}
```

Furthermore, we can fanout such a sink operation, since channels are thread-safe:
```
func work(n int, c <-chan T2) {
	var wg sync.WaitGroup
	wg.Add(n)

	for i := 0; i < n; i++ {
		go func() {
			sink(c)
			wg.Done()
		}()
	}

	wg.Wait()
}
```

If our sink is performing some sort of intensive I/O with a client, such as logging or inserting multiple records to a database, We may want to process our last stage in batches, i.e. `T2[]`.
We can refactor our sink code to write batches of messages, and to flush based on capacity and time:
```
func writeBatch([]T2) error { ... }

func sink(c <-chan T2) {

	const (
		bufSize = 50
		flushPeriod = time.Second * 4
	)

	buf := make([]T2, 0, bufSize)

	// alert when buffer should be cleared
	tick := time.Tick(d.flushPeriod)

	for {
		select {

		case <-tick:
			if len(buf) > 0 {
				if err := writeBatch(buf); err != nil {
					log.Println(err)
					return
				}
				buf = nil // reset buffer.
			}

		case elem, ok := <-c:
			if !ok {
				log.Print("no more work")
				return
			}

			buf = append(buf, elem)

			// flush if buffer capacity reached
			if len(buf) == bufSize {
				if err := writeBatch(buf); err != nil {
					log.Println(err)
					return
				}
				buf = nil // reset buffer.
			}
		}
	}
}
```

### An aside, Migrating From AKKA 

I recently migrated Scala services that heavily used AKKA Actors & Streams to Go services. 
These examples show how we can achieve stream processing semantics entirely by using the primitives and native libraries in Go.
