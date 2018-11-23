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

更新项目。需要考虑是否有代码更新，发送`_on_get_info`到scheduler2fetcher队列。

#### 2. `_check_task_done`

检查状态队列。如果是`_on_get_info`任务就更新项目的配置（`['min_tick', 'retry_delay', 'crawl_config']`），其它合法任务则会标记任务完成情况，更新项目状态信息。

- The `min_tick` is the greatest common divisor(GCD) of the interval of cronjobs. This value would be queried by scheduler when the project initial loaded. Scheudler may only send `_on_cronjob` task every `min_tick seconds`. It can reduce the number of tasks sent from scheduler.(`pyspider.libs.base_handler:106`)

#### 3. `_check_request`

取出任务进行处理。新任务直接进入任务队列（TaskQueue）等待处理。命中已有任务的话，使用`pyspider/scheduler/scheduler.py:822的on_old_request`进行处理。

逻辑：
- 请求正在处理的话，先不做处理，延期后再处理；
- 修改itag/一轮调度结束/强制重启，都会让task重启；
- cancel选项可以删除任务；

#### 4. `_check_cronjob`

`_last_tick`按秒增加，直到赶上当前时间，停止检查。

每次检查，遍历所有项目，看看项目是否（活跃<运行或调试状态>，`waiting_get_info`，`min_tick`存在，且`_last_tick`是`min_tick`的倍数，那么发送`_on_cronjob`任务到sheduler2fetcher队列

#### 5. `_check_select`

安排task到scheduler2fetcher队列

#### 6. `_check_delete`

删除保持状态为stop至少24h的任务，并且group_name中有delete字样就行

#### 7. `_try_dump_cnt`

每隔60秒，保存所有任务状态计数到本地文件

### 2. Fetcher



### 3. Processor

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