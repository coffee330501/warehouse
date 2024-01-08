参考：

[Selenium安装WebDriver：ChromeDriver谷歌浏览器驱动下载安装与使用最新版118/119/120](https://blog.csdn.net/nings666/article/details/134314452)



安装selenium

> pip install selenium

安装chrome驱动

[地址](https://googlechromelabs.github.io/chrome-for-testing/)

![image-20240104143503640](https://raw.githubusercontent.com/coffee330501/warehouse/master/pig/image-20240104143503640.png)

下载后解压到chrome安装路径下（默认安装路径: C:\Program Files\Google\Chrome\Application）

![image-20240104143822027](https://raw.githubusercontent.com/coffee330501/warehouse/master/pig/image-20240104143822027.png)

设置环境变量，chrome的安装路径



测试：

```python
from selenium import webdriver

if __name__ == '__main__':
    # 启动浏览器
    driver = webdriver.Chrome()
    driver.get("https://www.baidu.com")
    # 打印网页title
    print(driver.title)
```

结果：

![image-20240104144926532](https://raw.githubusercontent.com/coffee330501/warehouse/master/pig/image-20240104144926532.png)