# pyspider 源码学习

个人学习用，打算直接在源码中注释学习:)



#### 四大组件：scheduler、fetcher、processor、result worker

- Scheduler：任务调度（是否是新任务，是否需要重跑）
- Fetcher：获取页面内容，交由Processor处理
- Processor：解析页面内容交由Scheduler/Result Worker
- Result Worker：结果处理，默认在resultdb

