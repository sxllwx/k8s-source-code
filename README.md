

# k8s.io/apimachinery/pkg/watch

```go

//有Buffer的channel在Buffer满了之后可以有两种行为:
//Drop所有的流量
//Wait等待流量被消化

type FullChannelBehavior int

const (
	WaitIfChannelFull FullChannelBehavior = iota
	DropIfChannelFull
)
```

```go
// 广播器，可以将每一个事件分发给每一个关注者
type Broadcaster struct {
	
	lock sync.Mutex

	watchers     map[int64]*broadcasterWatcher
	// 下一个watcher的id
	nextWatcher  int64
	
	// TODO
	distributing sync.WaitGroup
    
	// 消息来源
	incoming chan Event

	// 每个watcher的通道的长度
	watchQueueLength int
	
	// If one of the watch channels is full, don't wait for it to become empty.
	// 如果一个watcher满了，不等待
	// Instead just deliver it to the watchers that do have space in their
	// channels and move on to the next event.
	// 而是分发给有空间的watcher的chan，然后就开始到了下一个事件了
	// It's more fair to do this on a per-watcher basis than to do it on the
	// "incoming" channel, which would allow one slow watcher to prevent all
	// other watchers from getting new events.
	// 这样做更加公平，而且应该在watcher端来做这个事情，这样作，可以防止一个慢的wacher阻止了，其他的watcher获取事件
	fullChannelBehavior FullChannelBehavior
}

```

```go
// NewBroadcaster creates a new Broadcaster. queueLength is the maximum number of events to queue per watcher.
// queueLength是每个watcher的最大的排队事件的长度
// It is guaranteed that events will be distributed in the order in which they occur,
// 这个广播器，可以保证事件是按照他发生的顺序进入watcher的观察队列的
// but the order in which a single event is distributed among all of the watchers is unspecified.
// 但是不保证单个事件在所有的watcher间的分发顺序
func NewBroadcaster(queueLength int, fullChannelBehavior FullChannelBehavior) *Broadcaster {
	m := &Broadcaster{
		watchers:            map[int64]*broadcasterWatcher{},
		incoming:            make(chan Event, incomingQueueLength),
		watchQueueLength:    queueLength,
		fullChannelBehavior: fullChannelBehavior,
	}
	m.distributing.Add(1)
	go m.loop()
	return m
}
```

```go
// loop receives from m.incoming and distributes to all watchers.
// 从incoming通道拿到消息，并且分发给每个watcher
func (m *Broadcaster) loop() {
	// Deliberately not catching crashes here. Yes, bring down the process if there's a
	// bug in watch.Broadcaster.
	for event := range m.incoming {
		if event.Type == internalRunFunctionMarker {
			event.Object.(functionFakeRuntimeObject)() // 这句话是重点，因为这个特殊的event.Type是被用来添加watcher的,
			continue
		}
		m.distribute(event)
	}
	m.closeAll()
	m.distributing.Done()
}


// distribute sends event to all watchers. Blocking.
// 分发给所有的watcher
func (m *Broadcaster) distribute(event Event) {
	// TODO  这里加锁的原因是什么?
	// **map 的并发读取都必须加上锁 (不然，一个goroutione写了map，另外一个读取的老的数据怎么办**
	m.lock.Lock()
	defer m.lock.Unlock()
	if m.fullChannelBehavior == DropIfChannelFull {
		for _, w := range m.watchers {
			select {
			case w.result <- event:
			case <-w.stopped: // 一般都是直接close这个stoped的channel，所以如果已经关闭，也能迅速进入这个case
			default: // Don't block if the event can't be queued.
			// 如果是channel满了就丢弃，直接drop不阻塞
			}
		}
	} else {
		for _, w := range m.watchers {
			select {
			case w.result <- event:
			case <-w.stopped:
			}
		}
	}
}


```


```go
// Execute f, blocking the incoming queue (and waiting for it to drain first).
// The purpose of this terrible hack is so that watchers added after an event
// won't ever see that event, and will always see any event after they are
// added.


// 先等待incoming的队列分发完， 然后在开始阻塞,执行f，
// 这样做的目的是为了让新加入watcher只能看到他加入之后到来的事件，而无法看到他加入之前发生的event
func (b *Broadcaster) blockQueue(f func()) {
	var wg sync.WaitGroup
	wg.Add(1)
	b.incoming <- Event{
		Type: internalRunFunctionMarker,
		Object: functionFakeRuntimeObject(func() {
			defer wg.Done()
			f()  // 这个f的计算发生在对方拿到event之后，执行这个方法
		}),
	}
	wg.Wait()
}

// Watch adds a new watcher to the list and returns an Interface for it.
// Note: new watchers will only receive new events. They won't get an entire history
// of previous events.

//  增加一个watcher，并且返回他的interface
//  该watcher只能收到他加入后之后 来的消息，而无法收到历史消息
//  为何不直接lock,unlock保护下，完成添加watcher,就是为了将watcher加入的时间和Event到来的时间点进行确认。
func (m *Broadcaster) Watch() Interface {
	var w *broadcasterWatcher
	m.blockQueue(func() {
		// 此处的lock也是为了保证map的并发安全（此处插一句话，golang除了chan，我暂时发现，哪个类型是并发安全的）
		m.lock.Lock() 
		defer m.lock.Unlock()
		id := m.nextWatcher
		m.nextWatcher++
		w = &broadcasterWatcher{
			result:  make(chan Event, m.watchQueueLength),
			stopped: make(chan struct{}),
			id:      id,
			m:       m,
		}
		m.watchers[id] = w
	})
	return w
}


// WatchWithPrefix adds a new watcher to the list and returns an Interface for it. It sends
// queuedEvents down the new watch before beginning to send ordinary events from Broadcaster.
// The returned watch will have a queue length that is at least large enough to accommodate
// all of the items in queuedEvents.

// 增加一个watcher，并且将传入的events， 首先发入该队列，并且在必要的时候加大该队列的buffer
func (m *Broadcaster) WatchWithPrefix(queuedEvents []Event) Interface {
	var w *broadcasterWatcher
	m.blockQueue(func() {
		m.lock.Lock()
		defer m.lock.Unlock()
		id := m.nextWatcher
		m.nextWatcher++
		length := m.watchQueueLength
		if n := len(queuedEvents) + 1; n > length {
			length = n
		}
		w = &broadcasterWatcher{
			result:  make(chan Event, length),
			stopped: make(chan struct{}),
			id:      id,
			m:       m,
		}
		m.watchers[id] = w
		for _, e := range queuedEvents {
			w.result <- e
		}
	})
	return w
}
```



```go

// stopWatching stops the given watcher and removes it from the list.
// 将一个给定id的watcher从wachter map中移除
func (m *Broadcaster) stopWatching(id int64) {
	// 加上锁来保护map
	m.lock.Lock()
	defer m.lock.Unlock()
	w, ok := m.watchers[id]
	if !ok {
		// No need to do anything, it's already been removed from the list.
		return
	}
	delete(m.watchers, id)
	close(w.result)
}

// closeAll disconnects all watchers (presumably in response to a Shutdown call).
// 断开所有的watcher, 一般在关闭的时候才用
func (m *Broadcaster) closeAll() {
	// 加上锁来保护map
	m.lock.Lock()
	defer m.lock.Unlock()
	for _, w := range m.watchers {
		close(w.result)
	}
	// Delete everything from the map, since presence/absence in the map is used
	// by stopWatching to avoid double-closing the channel.
	
	// TODO  上面说了个什么鬼。。完全没听懂
	m.watchers = map[int64]*broadcasterWatcher{}
}

```


```go
// Action distributes the given event among all watchers.
// 发送消息
func (m *Broadcaster) Action(action EventType, obj runtime.Object) {
	m.incoming <- Event{action, obj}
}
```

```go

// Shutdown disconnects all watchers (but any queued events will still be distributed).
// You must not call Action or Watch* after calling Shutdown. This call blocks
// until all events have been distributed through the outbound channels. Note
// that since they can be buffered, this means that the watchers might not
// have received the data yet as it can remain sitting in the buffered
// channel.

// 关闭所有的watcher，但是在排队的事件仍然会被分发，关闭之后，不能再调用Action,或者Watch*，这个调用会阻塞到所有的事件被分发完毕，
// 注意，为因为他们可以被buffer,所以这意味着这watcher，可能不会收到数据，因为他们可能被扔在buffer的chan中
func (m *Broadcaster) Shutdown() {
	close(m.incoming)
	m.distributing.Wait()
}
```
