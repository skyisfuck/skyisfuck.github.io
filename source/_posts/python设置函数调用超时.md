---
title: python设置函数调用超时
date: 2018-01-23 16:32:11
tags:
---
####  <font color=green>一、设置函数调用超时</font>
```python

import sys
import time
import signal
import subprocess


def cmd():
    def handler(signum, frame):             #定义信号处理函数
        raise AssertionError
    CMD = sys.argv[1]
    i = 0
    for i in (1, 3, 60):
        try:
            signal.signal(signal.SIGALRM, handler)                 #(信号,信号处理函数)
            signal.alarm(4)             #(设置4秒闹钟，函数调用超时4秒则raise AssertionError)
            sub = subprocess.Popen(CMD, shell=True, stdout=subprocess.PIPE)
            ret, err = sub.communicate()
            print ret
            print err
            print 'ok'
            break
        except AssertionError:
            print "%d timeout"%(i)
cmd()
```
说明：
1. 通过执行python test.py "sleep 5"来模拟超时
2. 引入signal模块，设置handler捕获超时信息，返回断言错误
3. alarm(4)，设置4秒闹钟，函数调用超时4秒则直接返回
4. 捕获异常，打印超时信息

```bash
[root@ftp python]# python test.py "sleep 3"
None
ok
[root@ftp python]# python test.py "sleep 4"
1 timeout
3 timeout
60 timeout
```
####  <font color=green>二、通过计时</font>
```python
#!/usr/bin/env python
import subprocess
import time
import sys


def command_run(command, timeout=10):
    for times in (10, 20, 30):
        proc = subprocess.Popen(command, stdout=subprocess.PIPE, shell=True)
        poll_seconds = .250
        deadline = time.time() + timeout
        while time.time() < deadline and proc.poll() == None:
            time.sleep(poll_seconds)
        if proc.poll() == None:
            if float(sys.version[:3]) >= 2.6:
                print "timeout"
                proc.terminate()

    ret,err = proc.communicate()
    print ret
    print err
    return ret, err

    # return stdout,stderr,proc.returncode
```
