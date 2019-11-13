---
title: Appium&Cucumber搭建UI自动化测试框架
date: 2019-04-05 15:52:28
tags:
	- UI自动化
	- 测试
---
## 背景知识

### WebDriver协议

一套抽象了页面行为的HTTP协议，Request表示UI操作，Response表示页面响应结果。该协议用JSON Body来描述命令和响应内容，是JSONWireProtocol的一种应用。

### Appium

一个基于WebDriver协议实现的自动化测试框架，主要针对移动端，支持原生App、Web App、Hybird App。

Appium总体是一个C/S结构，即Appium Server和Appium Client。Appium Server是一个暴露一系列API的Node Server（npm install -g appium），它以HTTP Response的形式响应来自Appium Client的不同请求。Appium Client本质就是一个HTTP Request发送器，它封装了一系列API来方便的发送各种UI操作请求。Appium的这种设计优点在于Client可以被灵活的选择，不管用何种语言，何种方式，只要能够发送符合协议的HTTP Request就可以充当Appium Client。

### Cucumber

一个实现BDD（Behavior-driven development）的框架。BDD暂不深究，我们主要关注Cucumber给自动化测试带来的改进。Cucumber为了良好的抽象`软件行为`，定义了一套DSL：Gherkin。使用Gherkin，我们可以将繁杂的自动化脚本进行拆分：将复杂的操作脚本沉淀为下层util，只暴露上层的自然语言的行为描述给开发者，大大降低脚本编写的门槛和成本。

## 工程设计

### 总体结构

![工程结构](https://i.loli.net/2019/11/13/kpZyntLc7ITqF5O.png)

工程总体是一个node工程，主要依赖了`webdriverio`，`cucumber`，`chai`三个npm包。 

<!--more-->

#### WebdricerIO

WebdriverIO是用Javascript实现的Appium Client，我们选用v4版本，因为v5暂不兼容Cucumber。

工程根目录的wdio.conf.js定义了WebdriverIO的初始化参数，主要定义当前Client要连接的Server地址及端口，需要连接的HTTP会话，测试脚本路径等信息。其中，配置文件里的capabilities数组代表当前Client需要开启的HTTP Session。每个capability对象代表一个Session，并定义了当前会话的测试目标，比如目标安装包路径、目标平台（Android/iOS）、目标测试设备等参数。详见[configurationfile](http://v4.webdriver.io/guide/testrunner/configurationfile.html)和[Capabilities](http://appium.io/docs/en/writing-running-appium/caps/)

#### Cucumber

配置文件里的cucumberOpts完成对Cucumber的配置，主要关注`require: ['./src/**/*.js']`参数，表示转译脚本的目录。

#### Chai

断言库，用来得出测试结果。

### 详细设计

#### 脚本目录

> /features

feature文件目录，feature文件即是Gherkin脚本文件，旨在使用一种`自然语言`描述一个可验证行为。

假如要测试一个点击按钮显示今天上不上班的App，那么我们的feature文件是这样的:

![feature](https://i.loli.net/2019/11/13/SvboPOmedsKTfr1.png)


蓝色文字是Gherkin的保留关键字

`今天是`、`我点击`、`显示`是页面行为，需要转译为Appium Client的API操作

单引号内的文字是输入参数

> /src

src存放**负责转译Gherkin**的js脚本和其他脚本

先看src/steps/*.js

该目录下的js文件负责对feature文件完成转译，以上面的feature为例，转译脚本如下：

```javascript
const { Given, When, Then } = require('cucumber');
const assert = require('chai').assert;

Given('今天是 {string}', function (string) {
    browser.setToday(string);
});

When('我点击 {string}', function (text) {
    browser.clickText(text);
});

Then('显示 {string}', function (string) {
    assert.equal(string, browser.getPageContent());
});
```

`Given`, `When`, `Then`分别对应`假如`，`当`，`那么`。

单引号里的字符串匹配feature中的`页面行为`和`入参`，最终进入后面的function里，使用browser对象（即Appium Client的全局对象）请求执行不同的UI操作，即完成了转译。

除了转译脚本外，还会沉淀一些utils到这个目录下，主要是对Client API的再次封装。比如查找元素的API: browser.element(string)，如果直接使用会经常因为页面未加载完而找不到元素，需要设置一个等待机制。utils.js便将封装好的方法内置到browser对象里，以方便上层调用。

Webdriverio Client的详细API文档见[http://v4.webdriver.io/api.html](http://v4.webdriver.io/api.html)

在配置文件里提到的`require: ['./src/*/.js']`参数，指向的就是转译js脚本的路径，因为这些脚本需要在执行feature文件之前被加载

#### Selector

UI自动化，核心是查找页面元素，因为几乎所有的页面操作（点击、输入、获取文本等）都要先准确地找到视图元素。webdriverio查找元素的方法：browser.element(string)只有一个入参，这个入参就叫selector，是一个字符串，它描述了Client希望找到一个或一组什么样的页面元素。所以，如何给目标元素编写selector是很关键的，它直接决定了我们的用例可执行与否。

官方的[Selector文档](http://v4.webdriver.io/guide/usage/selectors.html)基本演示了各种定位元素的策略。需要注意的是，针对原生控件，推荐使用各自平台的Selectors格式（UiSelector/UIATarget），以达到最高的成功率。前端页面则可做到平台无关。这意味着，我们可以在src目录下沉淀不同平台的Selector Generator，方便上层调用。

另外强调一个点，如何方便、迅速、优雅的查找元素，`不一定只有优化selector一条路可以走，适当的从规范控件设计，页面结构等方向入手，往往可以事半功倍`。总之是一个平衡的问题。

## 后续

有几个方向会有拓展价值和必要

1. 不断沉淀基础操作的js脚本，越来越多的写feature，越来越少的写js
2. 稳定性保证，retry机制
3. 错误捕获等hook操作，现已支持错误页面截图
4. 集成CI
5. 测试服务机搭建，多设备，多任务执行