[pyspider官网文档](https://docs.pyspider.org/en/latest/#installation)

## 安装

- `pip install pyspider`
- 运行命令`pyspider`，访问 http://localhost:5000/

> 报错

```bash
 Installing build dependencies ... done
  Getting requirements to build wheel ... error
  error: subprocess-exited-with-error

  × Getting requirements to build wheel did not run successfully.
  │ exit code: 10
  ╰─> [1 lines of output]
      Please specify --curl-dir=/path/to/built/libcurl
      [end of output]

  note: This error originates from a subprocess, and is likely not a problem with pip.
error: subprocess-exited-with-error

× Getting requirements to build wheel did not run successfully.
│ exit code: 10
╰─> See above for output.

note: This error originates from a subprocess, and is likely not a problem with pip.
```

去[官网](https://www.lfd.uci.edu/~gohlke/pythonlibs/)下载缺少的依赖

搜索`libcurl`，因为本地python版本为3.11.7所以下载cp311的

![image-20231230151852301](https://raw.githubusercontent.com/coffee330501/warehouse/master/pig/image-20231230151852301.png)

安装缺少的依赖 `pip install E:/Download/pycurl-7.45.1-cp311-cp311-win_amd64.whl`

再执行安装pyspider `pip install pyspider`

> 又报错

```bash
Traceback (most recent call last):
  File "<frozen runpy>", line 198, in _run_module_as_main
  File "<frozen runpy>", line 88, in _run_code
  File "C:\Users\dell\AppData\Local\Programs\Python\Python311\Scripts\pyspider.exe\__main__.py", line 4, in <module>
  File "C:\Users\dell\AppData\Local\Programs\Python\Python311\Lib\site-packages\pyspider\run.py", line 231
    async=True, get_object=False, no_input=False):
    ^^^^^
SyntaxError: invalid syntax
```

是因为spider中使用了3.7之后新加的关键字`async`，更换python版本为[3.6.8](https://www.python.org/ftp/python/3.6.8/python-3.6.8-amd64.exe)

重新安装`pip`

卸载 pip `pip uninstall pip`

安装 pip `python -m pip install --upgrade pip`

> 报错

```bash
Requirement already satisfied: pip in c:\users\dell\appdata\local\programs\python\python36\lib\site-packages
```

增加target参数再安装

`python -m pip install --upgrade pip --target=c:\users\dell\appdata\local\programs\python\python36\lib\site-packages`

> 还是报错

找不到pip命令

需要将python的script文件夹添加到环境变量中，可以通过`where python`查找python路径

还需要重新安装[版本对应的libcurl](https://download.lfd.uci.edu/pythonlibs/archived/cp36/pycurl-7.43.0.4-cp36-cp36m-win_amd64.whl)

安装pyspider`pip install pyspider`，执行`pyspider`

> 还是报错

```bash
TypeError: Can't instantiate abstract class ScriptProvider with abstract methods get_resource_inst
```

原因：wsgidav版本过高

执行命令 `pip install wsgidav==2.4.1`

再运行pyspider

> 还是报错

```bash
ImportError: cannot import name 'DispatcherMiddleware'
```

原因：werkzeug版本过高

执行命令 `pip install werkzeug==0.16.1`

为防止一样类型的错误，再执行 `pip install flask==1.0`

运行 `pyspider`

> 成功启动

```bash
[I 231230 16:05:58 processor:211] processor starting...
[I 231230 16:05:58 scheduler:647] scheduler starting...
[I 231230 16:05:58 scheduler:586] in 5m: new:0,success:0,retry:0,failed:0
[I 231230 16:05:58 tornado_fetcher:638] fetcher starting...
[I 231230 16:05:58 scheduler:782] scheduler.xmlrpc listening on 127.0.0.1:23333
[I 231230 16:05:58 app:76] webui running on 0.0.0.0:5000
```

启动地址: localhost:5000