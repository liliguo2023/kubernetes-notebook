# 								backoff	

## Backoff

````
type backoffEntry struct {
	backoff    time.Duration	// 退避时间
	lastUpdate time.Time			// 最后一次更新时间
}

type Backoff struct {
	sync.RWMutex
	Clock           clock.Clock
	defaultDuration time.Duration								// 默认退避时间
	maxDuration     time.Duration								// 最大退避时间
	perItemBackoff  map[string]*backoffEntry		// 独立存储退避时间
	rand            *rand.Rand									// 用于和maxJitterFactor一起生成抖动时间

	// maxJitterFactor adds jitter to the exponentially backed off delay.
	// if maxJitterFactor is zero, no jitter is added to the delay in
	// order to maintain current behavior.
	maxJitterFactor float64
}

func newBackoff(clock clock.Clock, initial, max time.Duration, maxJitterFactor float64) *Backoff {
	var random *rand.Rand
	if maxJitterFactor > 0 {
		random = rand.New(rand.NewSource(clock.Now().UnixNano()))
	}
	return &Backoff{
		perItemBackoff:  map[string]*backoffEntry{},
		Clock:           clock,
		defaultDuration: initial,
		maxDuration:     max,
		maxJitterFactor: maxJitterFactor,
		rand:            random,
	}
}
````



````go
// Get the current backoff Duration
// 根据id获取退避时间
func (p *Backoff) Get(id string) time.Duration {
	p.RLock()
	defer p.RUnlock()
	var delay time.Duration
	entry, ok := p.perItemBackoff[id]
	if ok {
		delay = entry.backoff
	}
	return delay
}
````



````go
// move backoff to the next mark, capping at maxDuration
func (p *Backoff) Next(id string, eventTime time.Time) {
	p.Lock()
	defer p.Unlock()
	entry, ok := p.perItemBackoff[id]
  // 如果不在退避列表，或者eventTime - 最后一次退避更新时间，大于2倍的maxDuration，则重新设置
	if !ok || hasExpired(eventTime, entry.lastUpdate, p.maxDuration) {
    // 初始化退避时间为 defaultDuration
		entry = p.initEntryUnsafe(id)
    // 如果设置了抖动时间，则加上抖动时间
		entry.backoff += p.jitter(entry.backoff)
	} else {
    // 指数级增长退避时间
		delay := entry.backoff * 2       // exponential
		delay += p.jitter(entry.backoff) // add some jitter to the delay
    // 最大不超过maxDuration
		entry.backoff = time.Duration(integer.Int64Min(int64(delay), int64(p.maxDuration)))
	}
	entry.lastUpdate = p.Clock.Now()
}

// Take a lock on *Backoff, before calling initEntryUnsafe
// 初始化Backoff
func (p *Backoff) initEntryUnsafe(id string) *backoffEntry {
	entry := &backoffEntry{backoff: p.defaultDuration}
	p.perItemBackoff[id] = entry
	return entry
}

// 计算抖动时间
func (p *Backoff) jitter(delay time.Duration) time.Duration {
	if p.rand == nil {
		return 0
	}

	return time.Duration(p.rand.Float64() * p.maxJitterFactor * float64(delay))
}

// After 2*maxDuration we restart the backoff factor to the beginning
func hasExpired(eventTime time.Time, lastUpdate time.Time, maxDuration time.Duration) bool {
	return eventTime.Sub(lastUpdate) > maxDuration*2 // consider stable if it's ok for twice the maxDuration
}
````



~~~~go
// Reset forces clearing of all backoff data for a given key.
// 格局id删除对应item
func (p *Backoff) Reset(id string) {
	p.Lock()
	defer p.Unlock()
	delete(p.perItemBackoff, id)
}
~~~~



~~~~go
// Returns True if the elapsed time since eventTime is smaller than the current backoff window
func (p *Backoff) IsInBackOffSince(id string, eventTime time.Time) bool {
	p.RLock()
	defer p.RUnlock()
	entry, ok := p.perItemBackoff[id]
	if !ok {
		return false
	}
	if hasExpired(eventTime, entry.lastUpdate, p.maxDuration) {
		return false
	}
	return p.Clock.Since(eventTime) < entry.backoff
}
~~~~



~~~~go
// Garbage collect records that have aged past maxDuration. Backoff users are expected
// to invoke this periodically.
// 退避时间超过两倍的maxDuration，进行删除
func (p *Backoff) GC() {
	p.Lock()
	defer p.Unlock()
	now := p.Clock.Now()
	for id, entry := range p.perItemBackoff {
		if now.Sub(entry.lastUpdate) > p.maxDuration*2 {
			// GC when entry has not been updated for 2*maxDuration
			delete(p.perItemBackoff, id)
		}
	}
}
~~~~

