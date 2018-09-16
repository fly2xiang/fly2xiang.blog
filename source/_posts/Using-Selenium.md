title: "使用 Selenium 与 ChromeDriver 进行自动化测试"
date: 2018-07-19 23:01:35
tags:
- Selenium
- 自动化测试
- Automated Testing
categories: 
- 自动化测试

---


## 概述

[Selenium](https://www.seleniumhq.org/) 提供了一系列控制浏览器的 API ，使用这些 API 可以方便的通过编程方式与浏览器进行交互，模拟用户操作，达到自动化测试的目的。

[ChromeDriver](http://chromedriver.chromium.org/) 提供了 Selenium 与 Chrome 浏览器交互的方式，ChromeDriver 实现了 [WebDriver Wire Protocol](https://github.com/SeleniumHQ/selenium/wiki/JsonWireProtocol)。

WebDriver 是一个用来做网页自动化测试 (Automated Testing) 的开源工具，提供了浏览网页、用户输入、执行 Javascript 等能力。

## 准备环境

Selenium 支持多种语言：Java、C#、Ruby、Python、Perl、Javascript，这里使用 Python。操作系统是 Windows 10 版本 1803。

首先从 [Python 官网](https://www.python.org) 下载 Python 3.7 安装文件并安装，安装过程中选择将 python 加入到 PATH 环境变量或安装后手动将 python 与 pip 所在目录加入环境变量。

从 ChromeDriver 官网下载 ChromeDriver 2.4 解压可执行文件，并将所在目录加入 PATH 环境变量。

打开 cmd 执行 `pip install selenium` 安装 Selenium。

执行 `python` 进入 Python 命令行，执行以下代码可以看到弹出了一个 Chrome 窗口，窗口地址栏下显示 “Chrome 正受到自动测试软件的控制”，表示安装成功。

```python
>>> from selenium import webdriver
>>> driver = webdriver.Chrome()
```

## 简单使用

一个简单的例子：使用百度搜索 Selenium。接着上面的代码在 Python 命令行中执行：

```python
>>> driver.get('https://www.baidu.com')
>>> driver.find_element_by_css_selector('#kw').send_keys('Selenium')
>>> driver.find_element_by_css_selector('#su').click()
```

上面的代码分别打开了百度首页、在搜索框中键入了 "Selenium" 单词、按下了搜索键。

## 主要 API

* `driver.title` 得到网页标题
* `driver.find_element(s)_by_...` 查找一个或多个元素，支持ID、类名、class、Link Text(链接中的文字)、CSS选择器、XPath
* `driver.send_keys` 输入文本
* `element.get_attribute` 得到元素属性
* `driver.execute_script` 执行 Javascript

更多的 API 可以参考[官方文档](https://www.seleniumhq.org/docs/03_webdriver.jsp#selenium-webdriver-api-commands-and-operations)

## 与 Javascript 交互

可以使用 `driver.execute_script` 与浏览器进行交互。

使用 Javascript 查找元素：

```python
>>> kw_input = driver.execute_script('return document.querySelector("#kw")')
```

使用 Javascript 设置元素属性：

```python
>>> driver.execute_script('arguments[0].value = "";', kw_input)
```

## 模拟手机端

* 使用 Chrome 内置的模拟设备来模拟，在 Chrome 的 Developer Tools -> Settings -> Devices 中可以看到内置的设备。

```python
mobileEmulation = {'deviceName': 'iPhone X'}
options = webdriver.ChromeOptions()
options.add_experimental_option('mobileEmulation', mobileEmulation)

driver = webdriver.Chrome(executable_path='chromedriver.exe', chrome_options=options)

driver.get('http://www.baidu.com')
```

* 自定义参数模拟

```python
WIDTH = 320
HEIGHT = 640
PIXEL_RATIO = 3.0
UA = 'Mozilla/5.0 (Linux; Android 4.1.1; GT-N7100 Build/JRO03C) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/35.0.1916.138 Mobile Safari/537.36 T7/6.3'

mobileEmulation = {"deviceMetrics": {"width": WIDTH, "height": HEIGHT, "pixelRatio": PIXEL_RATIO}, "userAgent": UA}
options = webdriver.ChromeOptions()
options.add_experimental_option('mobileEmulation', mobileEmulation)

driver = webdriver.Chrome(executable_path='chromedriver.exe', chrome_options=options)
driver.set_window_size(WIDTH,HEIGHT)
driver.get('http://www.baidu.com')
```

## 参考链接

[Selenium Documentation](https://www.seleniumhq.org/docs/03_webdriver.jsp#selenium-webdriver-api-commands-and-operations)
[Selenium-Python 中文文档](https://python-selenium-zh.readthedocs.io/zh_CN/latest)
