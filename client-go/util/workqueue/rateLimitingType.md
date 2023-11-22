# 								rateLimitingType

````go
// 对延迟队列和限流器的封装
type rateLimitingType struct {
	DelayingInterface

	rateLimiter RateLimiter
}

// AddRateLimited AddAfter's the item based on the time when the rate limiter says it's ok
// 加到延迟队列之前，先通过限流器计算延迟时间
func (q *rateLimitingType) AddRateLimited(item interface{}) {
	q.DelayingInterface.AddAfter(item, q.rateLimiter.When(item))
}

// 返回失败次数
func (q *rateLimitingType) NumRequeues(item interface{}) int {
	return q.rateLimiter.NumRequeues(item)
}

// 从失败列表删除
func (q *rateLimitingType) Forget(item interface{}) {
	q.rateLimiter.Forget(item)
}
````

