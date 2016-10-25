+++
date = "2016-10-08T22:28:53+08:00"
draft = false
title = "logging"
thumbnail = "images/python/python.jpg"

categories = ["python"]

+++

# 一个简单的例子
    import logging
    logging.warning('Watch out!')  # 输出到控制台
    logging.info('I told you so')  # 什么都不输出
输出:

    WARNING:root:Watch out!

logging默认输出到控制台，默认输出级别为WARNING，所以INFO级别没有输出。

# 异常处理
logging为异常专门提供了`logger.exception()`接口以便输出异常，该接口只应该在异常处理时使用。

    try:
        1/0
    except:
        logging.exception('Excepion:')

输出:

    ERROR:root:Excepion:
    Traceback (most recent call last):
      File "<stdin>", line 2, in <module>
    ZeroDivisionError: integer division or modulo by zero

# 配置日志
    logging.basicConfig(filename='example.log',level=logging.DEBUG)
basicConfig应该在记录日志之前被调用，可以被调用多次，但只有第一次调用会生效。

# 高级话题
logging模块提供了loggers, handlers, filters, and formatters更加现代的组件:

* Loggers:    暴露接口以便应用程序直接调用
* Handlers:   将日志发送到相应目的地
* Filters:    对日志进行过滤
* Formatters: 指定日志的输出格式

日志的传输方向为: looggers->handlers->filters->formatters。

logger在概念上处于一个树形结构的名字空间。例如：scan是scan.text、scan.html和scan.pdf的父亲。logger的名字可以是任意字符串，但通常使用模块名:

    logger = logging.getLogger(__name__)

logger的根结点的名字为root:

    logger = logging.getLogger() 等价于 logger = logger.getLogger('')

# 日志流
{{% img src="images/python/logging_flow.png" %}}

# Loggers
logger对象有三个任务。

1. 导出接口以便应用程序进行调用
1. 通过filter对日志进程过滤
1. 将日志转发到感兴趣的logger对象

child loggers会转发日志到ancestor loggers，由属性`logger.propagate`进行控制。鉴于此，通常只在root logger里设置handlers。

# 参考资料
* [官方文档-Python2](https://docs.python.org/2/library/logging.html)
* [官方文档-Python3](https://docs.python.org/3/library/logging.html)
