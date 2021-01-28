# Python on MR

[TOC]

将近期写的MR程序及过程记录下来。
简单介绍下环境：

```
hadoop2.6.4
hadoop-streaming-2.6.0.jar
线上python2，线下python3都可以用123
```

------

**首先放上需要的代码，定制python代码，很爽**
## mapper.py

```
#!/usr/bin/python
# -*- coding: UTF-8 -*-
import sys
#定义一个函数读标准输入或者文件内容，读入内容按空格分割
def read(file):
    for line in file:
        #yield可以使一个函数具有迭代功能，也可以起到缓冲作用
        yield line.split()

#定义主函数，得到输入中每个词的个数，可重复出现
def main(separator=','):
    data = read(sys.stdin)
    for words in data:
        for word in words:
            print ('%s%s%d' % (word, separator, 1))

if __name__ == "__main__":
    main()123456789101112131415161718
```

## reduce.py

```
#!/usr/bin/python
# -*- coding: UTF-8 -*-
from itertools import groupby
from operator import itemgetter
import sys
#定义函数读入数据（来自mapper输出结果，mapper.py结果怎样与本读入对接由hadoop-streaming.jar处理）并且按逗号分隔
def read(file, separator=','):
    for line in file:
        yield line.rstrip().split(separator, 1)
#定义主函数 
def main(separator=','):
    data = read(sys.stdin, separator=separator)
    #groupby(a,b)有迭代功能，按b为a分组，a有序。
    for current_word, group in groupby(data, itemgetter(0)):
        try:
            total_count = sum(int(count) for current_word, count in group)
            print ("%s%s%d" % (current_word, separator, total_count))
        except ValueError:
            pass

if __name__ == "__main__":
    main()

123456789101112131415161718192021222324
```

## **然后将代码上传到服务器测试**
用的xshell ，要么直接拖拽，要么rz命令上传，略过。

```
cat test.dat |python /opt/jskp/jinjiwei/MR/mapper.py|sort|python /opt/jskp/jinjiwei/MR/reducer.py1
```

上个图![这里写图片描述](https://img-blog.csdn.net/20180813143416473?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI5MTg2MTk5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

**没问题本地服务器测试数据上传到hdfs**
上传的话下面两个命令都可以：

```
hdfs dfs -copyFromLocal MR/test.dat hdfs:///JJW/
#-copyFromLocal只能拷贝本地文件到HDFS中限制严格。
hdfs dfs -put MR/test.dat hdfs:///JJW/
#可以把本地或者HDFS上的文件拷贝到HDFS中。1234
```

**查看一下没问题：**
![这里写图片描述](https://img-blog.csdn.net/2018081314420018?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI5MTg2MTk5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
**简单配置一下看看结果**：

```bash
hadoop jar ../hadoop-streaming-2.6.0.jar 
-mapper 'python mapper.py' -file ./mapper.py 
-reducer 'python reducer.py' -file reducer.py 
-input /JJW/test.dat
-output /JJW/output112345

## ambari version
hadoop jar /usr/hdp/3.0.1.0-187/hadoop-mapreduce/hadoop-streaming-3.1.1.3.0.1.0-187.jar \
-mapper 'python mapper.py' -file ./mapper.py \
-reducer 'python reduce.py' -file ./reduce.py \
-input /tmp/test.dat   \
-output /tmp/outputjz1
```

**最后一行是这么个结果就成功了：**
![这里写图片描述](https://img-blog.csdn.net/20180813144535818?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI5MTg2MTk5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
**然后看一下输出结果：**
![这里写图片描述](https://img-blog.csdn.net/201808131447351?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI5MTg2MTk5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
**查看输出内容：**
![这里写图片描述](https://img-blog.csdn.net/20180813145014139?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI5MTg2MTk5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
与本地服务器测试结果一致，至此MR小demo结束。
`插个小段子，如果运行hadoop命令总是报错，去喝杯咖啡或者睡一觉，你会发现出问题的地方很智障。`

**参考资料**
[Writing An Hadoop MapReduce Program In Python](http://www.michael-noll.com/tutorials/writing-an-hadoop-mapreduce-program-in-python/)
[Hadoop Streaming 使用及参数设置](https://www.cnblogs.com/hopelee/p/7476145.html)