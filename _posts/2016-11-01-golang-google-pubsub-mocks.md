---
layout: post
title:  "Go Testing: Mocks for Google Cloud Pub/Sub"
date:   2016-11-01 00:00:00
categories: golang data streams functional programming
---

I recently noticed a github [issue](https://github.com/GoogleCloudPlatform/google-cloud-go/issues/409) which pleads for publicly exposed interfaces to aid in testing and dep injection of Google Cloud Go libraries.

Heeding the advice of the Google team, I use `namespaces` in order to run isolated tests on services that are backed by Cloud Datastore. 
In my CircleCI test, this is implemented with linked docker containers (1 for my application and 1 for the Gcloud emulator). 
This has worked well for me, especially given the complexity of a caching layer over some transactions.

Nevertheless, it's a little simpler to write tests for Cloud Pub/sub if we create our own Go interface.

### Reading Messages

One can find great examples of using the Pub/Sub API [sometimes shrouded within GoDocs](https://godoc.org/cloud.google.com/go/pubsub).

To read messages off of Pubsub, we might do something like this:

``` 
import (
	"context"
	"log"

	ps "cloud.google.com/go/pubsub"
	"google.golang.org/api/iterator"
)

const N = 10

func pull(ctx context.Context, sub *ps.Subscription) <-chan *ps.Message {

	out := make(chan *ps.Message)

	go func() {
		it, err := sub.Pull(ctx)
		if err != nil {
			log.Printf("error pulling: %v", err)
			return
		}
		defer it.Stop()

		for i := 0; i < N; i++ {
			msg, err := it.Next()
			if err == iterator.Done {
				break
			}
			if err != nil {
				// handle error somehow
				log.Printf("error pulling: %v", err)
				break
			}

			out <- msg
		}
	}()

	return out
}
```

The function pulls up to `N` messages asynchronously and returns a channel that is closed after no more messages are available, or an error occurs.
The messages aren't ACKed, and that is left to the consumer of the channel by calling `msg.Done(true)`.

### Refactoring with Interfaces 

The ubiquity of such an operation lends itself nicely to writing an interface:

```
type puller interface {
	Pull() <-chan envelope
	Size() int
}

type envelope interface {
	Id() string
	Data() []byte
	Ack()
}
```

We've defined an `envelope` interface so that we can acknowledge messages that have been successfully processed, and avoid the RPC call in `Done(bool)` for tests.
Below are both "real" and "fake" implementations to use in our application, and tests, respectively:

```
type Puller struct {
	sub *ps.Subscription
	n   int
	ctx context.Context
}

func New(ctx context.Context, sub *ps.Subscription, size int) *Puller {
	return &Puller{
		sub: sub,
		n:   size,
		ctx: ctx,
	}
}

type Envelope struct {
	*ps.Message
}

func (e Envelope) Id() string   { return e.ID }
func (e Envelope) Data() []byte { return e.Message.Data }
func (e Envelope) Ack()         { e.Done(true) }

func (p *Puller) Size() int { return p.n }

func (p *Puller) Pull() <-chan envelope {
	out := make(chan envelope)

	go func() {
		it, err := p.sub.Pull(p.ctx)
		if err != nil {
			log.Printf("error pulling: %v", err)
			return
		}
		defer it.Stop()

		for i := 0; i < p.Size(); i++ {
			msg, err := it.Next()
			if err == iterator.Done {
				break
			}
			if err != nil {
				// handle error somehow
				log.Printf("error pulling: %v", err)
				break
			}

			out <- Envelope{msg}
		}
	}()

	return out
}

type fakePuller struct {
	results []*fakeEnvelope
}

type fakeEnvelope struct {
	data []byte
	ID   string
}

func (e *fakeEnvelope) Id() string   { return e.ID }
func (e *fakeEnvelope) Data() []byte { return e.data }
func (e *fakeEnvelope) Ack()         {}

func (p *fakePuller) Size() int { return len(p.results) }

func (p *fakePuller) Pull() <-chan envelope {
	out := make(chan envelope)
	for i := 0; i < p.Size(); i++ {
		out <- p.results[i]
	}

	return out
}
```

### Usage 

In our application: 

```
func CountWords(p puller) (num int) {
	for sentence := range p.Pull() {
		num += len(strings.Split(string(sentence.Data()), " "))
	}
	return
}
```

and in our tests:

```
func TestCountWords(t *testing.T) {
	expected := 5

	pulled := []*fakeEnvelope{
		&fakeEnvelope{data: []byte(`hello world`)},
		&fakeEnvelope{data: []byte(`foo bar`)},
		&fakeEnvelope{data: []byte(`bar`)},
	}
	mock := &fakePuller{pulled}

	got := CountWords(mock)
	if expected != got {
		t.Errorf("TestProcess; expected %d; got %d;", expected, got)
	}
}
```




