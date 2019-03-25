http://docs.pyspider.org/en/latest/

# pyspider 源码学习

个人学习用，打算直接在源码中注释学习:)

项目结构：
```shell
|____.github/  # 可以放入issue模板。https://help.github.com/articles/creating-an-issue-template-for-your-repository/
|____data/
|____docs/
|____pyspider/
|____tests/
|____tools/
|____.coveragerc
|____.gitignore
|____.travis.yml
|____Dockerfile
|____MANIFEST.in  # 打包所额外需要的文件 https://docs.python.org/3/distutils/sourcedist.html#specifying-the-files-to-distribute
|____LICENSE
|____mkdocs.yml
|____README.md
|____requirements.txt
|____run.py
|____setup.py  # 安装配置 http://yansu.org/2013/06/07/learn-python-setuptools-in-detail.html
|____tox.ini
```

### 四大组件：scheduler、fetcher、processor、result worker

- Scheduler：任务调度（是否是新任务，是否需要重跑）
- Fetcher：获取页面内容，交由Processor处理
- Processor：解析页面内容交由Scheduler/Result Worker
- Result Worker：结果处理，默认在resultdb


## 流程

- 启动phantomjs
- 启动result worker，使用队列process2result，监听process返回的结果，入库
- 启动processor
  1. 从fetcher2processor取task和response
  2. 调用callback对应的函数处理response(**参见pyspider.libs.base_handler.py:176**)，将处理结果放入processor2result，将运行状态放入status_queue，如果有后续任务，放入newtask_queue
- 启动fetcher
  1. 从scheduler2fetcher取出task，获得url，决定用什么抓取（http、data入库、phantomjs、splash，**参见pyspider.fetcher.tornado_fetcher.py:132**），结果(task, result)入fetcher2processor
- 启动scheduler
  1. 从projectdb拿到active(running or debuging)的project，通过project从taskdb拿到task放入project.task_queue
  2. 对_on_get_info的task进行更新
  3. 将首任务on_cronjob放入scheduler2fetcher
  4. 将结束任务on_finished和其它任务放入scheduler2fetcher
  5. 检查已经标为删除的任务，将其删除
  6. 每60s，保存计数器到文件
- 启动webUI

## 详细

### 1. Scheduler

#### 1. `self._update_projects()`

检测projectdb，更新项目到scheduler的projects列表。需要考虑是否有代码更新，发送`_on_get_info`到scheduler2fetcher队列。

#### 2. `_check_task_done`

检查状态队列。如果是`_on_get_info`任务就更新项目的配置（`['min_tick', 'retry_delay', 'crawl_config']`），其它合法任务则会标记任务完成情况，更新项目状态信息，失败任务会放入`task_queue`重试。

- The `min_tick` is the greatest common divisor(GCD) of the interval of cronjobs. This value would be queried by scheduler when the project initial loaded. Scheudler may only send `_on_cronjob` task every `min_tick seconds`. It can reduce the number of tasks sent from scheduler.(`pyspider.libs.base_handler:106`)

#### 3. `_check_request`

从`newtask_queue`取出任务进行处理。新任务直接进入任务队列（TaskQueue）等待处理。命中已有任务的话，使用`pyspider/scheduler/scheduler.py:822的on_old_request`进行处理。

逻辑：
- 请求正在处理的话，先不做处理，延期后再处理；
- 修改itag/一轮调度结束/强制重启，都会让task重启；
- cancel选项可以删除任务；

#### 4. `_check_cronjob`

`_last_tick`按秒增加，直到赶上当前时间，停止检查。

每次检查，遍历所有项目，看看项目是否（活跃<运行或调试状态>，`waiting_get_info`，`min_tick`存在，且`_last_tick`是`min_tick`的倍数，那么发送`_on_cronjob`任务到sheduler2fetcher队列

- min_tick的生成
  - `pyspider.libs.base_handler.every`，使用every装饰，会补上tick属性。然后`pyspider.libs.base_handler.BaseHandlerMeta`装饰后的basehandle会就算最大公因数。这样调度的时候只要是GCD的倍数，说明就该调度了

#### 5. `_check_select`

安排task到scheduler2fetcher队列

#### 6. `_check_delete`

删除保持状态为stop至少24h的项目，并且group_name中有delete字样就行

#### 7. `_try_dump_cnt`

每隔60秒，保存所有项目状态计数到本地文件

### 2. Fetcher

#### 1. `async_fetch`

获取url内容，send_result作为回调发送数据到fetcher2processor

- url开头是`data:`使用`data_fetch`
  - 不做请求处理，直接装配成结果
- 配置里面有js或者phantomjs，用`phantomjs_fetch`
- 配置里面有splash用`splash_fetch`
  - 和phantomjs一样，都是把请求转发给对应的处理代理，然后拿到响应信息
- 其余用`http_fetch`

调用on_result更新计数器

### 3. Processor

- 通过项目名字，去项目db拿到项目，然后`pyspider.processor.project_module.ProjectManager#build_module`，构建脚本对象。
  ```
  return {
            'loader': loader,  # 项目loader
            'module': module,
            'class': _class,  # handler类
            'instance': instance,  # handler实例
            'exception': None,
            'exception_log': '',
            'info': project,
            'load_time': time.time(),
        }
  ```
- 拿到handler去执行脚本`project_data['instance'].run_task`
  - 调用`_run_func`执行对应的回调函数
  - inspect.getargspec可以获取参数名字，不错不错
- 执行`on_result`处理上一步的结果
  - 放入result队列

### 4. Result worker

这一部分主要就是调用resultdb进行result的save工作。`pyspider/database`下有各种数据库的实现。

循环监听部分的异常处理挺有意思的，考虑了keyboard和assert的错误。
```
try:
    task, result = self.inqueue.get(timeout=1)
    self.on_result(task, result)
except Queue.Empty as e:
    continue
except KeyboardInterrupt:
    break
except AssertionError as e:
    logger.error(e)
    continue
except Exception as e:
    logger.exception(e)
    continue
```

### 5. Phantomjs

官方文档：http://phantomjs.org/documentation/

调用`phantomjs_fetcher.js`进行页面解析，其中主要用了phantomjs自带的webpage和webserver模块

- *refer*
  1. [PhantomJS](http://javascript.ruanyifeng.com/tool/phantomjs.html)
  2. [简介,phantomjs,中文,文教,教程,Node.js开发社区 ](https://www.ctolib.com/docs-PhantomJS-c-index.html)

## 番外

### 1. `pyspider/libs/utils.py`


## 提问

#### 1. scheduler是如何分发的？

依靠3个优先队列，分别为`priority_queue、time_queue、processing`。

从`time_queue、processing`取出到执行时间的任务放入`priority_queue`；

`processing queue`的`重试时间=当前执行时间+超时时间`，一个任务结束时会删除该队列对应的task(refer:`pyspider.scheduler.task_queue.TaskQueue#done`)，如果到重试时间该任务还在，说明超时了，该重试。

#### 2. 限流是怎么做的？

`pyspider.scheduler.token_bucket.Bucket`

每次从令牌桶中get，然后desc

#### 3. 如何控制任务的执行周期？



#### 4. 错误停止机制是如何的？



#### 5. 错误重试怎么进行的？

processor将fetch失败的结果发送到`status_queue`，然后在scheduler处理

- `pyspider.fetcher.tornado_fetcher.Fetcher#send_result` 会把所有的请求结果都发送给processor
  - 错误的result会封装成这样：`pyspider.fetcher.tornado_fetcher.Fetcher#handle_error`
- processor不论任务是什么状态码都会执行，然后发送到`status_queue`，`pyspider/processor/processor.py:169`
- scheduler在`pyspider.scheduler.scheduler.Scheduler#_check_task_done`处理`status_queue`的结果
  - `pyspider.scheduler.scheduler.Scheduler#on_task_status`下的`on_task_failed`处理错误task

#### 6. 如何在web页面进行编辑，选择元素路径的

#### 7. scheduler为什么只能单点？

- 需要维护项目列表，以及对应项目所对应的`task_queue`

