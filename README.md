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

#### 四大组件：scheduler、fetcher、processor、result worker

- Scheduler：任务调度（是否是新任务，是否需要重跑）
- Fetcher：获取页面内容，交由Processor处理
- Processor：解析页面内容交由Scheduler/Result Worker
- Result Worker：结果处理，默认在resultdb


### 流程

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