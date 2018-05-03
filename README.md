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

