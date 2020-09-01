title: HttpRunner的尝试
author: James
tags:

  - 测试
  - httprunner
categories:
  - 测试
date: 2019-08-15 16:02:00

---


# 目的

最近工作职位发生了变化，被任命为公司的技术总监，负责管理后端、运维、客户端、测试团队。通过一段时间的观察，团队的当前主要短板：*测试交付延期，覆盖率不足*； 仔细分析了下原因：整个测试团队目前还处于初级阶段（虽然公司已经发展了5年了），每个人还用excel写测试点，接口测试和自动化测试也从未开展。 团队小伙伴的理由都是任务重所以没有时间学习。 为了更好的提升测试团队的工作效率，并促进成员的成长， 故而决定以身作则，自己抽时间来学习整个测试体系，包括接口、自动化测试、相关工具(testlink、jmeter、jira、httprunner)。 ***httprunner***是在学习和调研过程中一个比较好测试框架；我也阅读了框架作者的blog，确实是一位值得尊敬和学习的测试专家。

本文将通过一个简单的实际示例来展示 HttpRunner 的核心功能使用方法。

# 简介

**HttpRunner** 是一款面向 HTTP(S) 协议的通用测试框架，只需编写维护一份 `YAML/JSON` 脚本，即可实现自动化测试、性能测试、线上监控、持续集成等多种测试需求。

<!-- more -->

# 为什么是它

- 继承 [Requests](http://docs.python-requests.org/en/master/) 的全部特性，轻松实现 HTTP(S) 的各种测试需求
- 采用 `YAML/JSON` 的形式描述测试场景，保障测试用例描述的统一性和可维护性
- 借助辅助函数（debugtalk.py），在测试脚本中轻松实现复杂的动态计算逻辑
- 支持完善的测试用例分层机制，充分实现测试用例的复用
- 测试前后支持完善的 hook 机制
- 响应结果支持丰富的校验机制
- 基于 HAR 实现接口录制和用例生成功能（[har2case](https://github.com/HttpRunner/har2case)）
- 结合 [Locust](http://locust.io/) 框架，无需额外的工作即可实现分布式性能测试
- 执行方式采用 CLI 调用，可与 Jenkins 等持续集成工具完美结合
- 测试结果统计报告简洁清晰，附带详尽统计信息和日志记录
- 极强的可扩展性，轻松实现二次开发和 Web 平台化

# 安装

## 运行环境

HttpRunner 是一个基于 Python 开发的测试框架，可以运行在 macOS、Linux、Windows 系统平台上。

**Python 版本**：HttpRunner 支持 Python 3.4 及以上的所有版本，并使用 Travis-CI 进行了[持续集成测试](https://travis-ci.org/HttpRunner/HttpRunner)，测试覆盖的版本包括 2.7/3.4/3.5/3.6/3.7。虽然 HttpRunner 暂时保留了对 Python 2.7 的兼容支持，但强烈建议使用 Python 3.4 及以上版本。

**操作系统**：推荐使用 macOS/Linux。

## 安装方式

HttpRunner 的稳定版本托管在 PyPI 上，可以使用 `pip` 进行安装。

```shell
$ pip install httprunner
```

如果你需要使用最新的开发版本，那么可以采用项目的 GitHub 仓库地址进行安装：

```shell
$ pip install git+https://github.com/HttpRunner/HttpRunner.git@master
```

安装验证运行

```shell
$ hrun -V
2.0.2
```

# 项目文件

在 HttpRunner 自动化测试项目中，主要存在如下几类文件：

- `YAML/JSON`（必须）：测试用例文件，存储接口测试相关信息

- ```
  debugtalk.py（可选）：存储项目中逻辑运算辅助函数
  ```

  - 该文件存在时，将作为项目根目录定位标记，其所在目录即被视为项目工程根目录
  - 该文件不存在时，运行测试的所在路径（`CWD`）将被视为项目工程根目录
  - 测试用例文件中的相对路径（例如`.csv`）均需基于项目工程根目录
  - 运行测试后，测试报告文件夹（`reports`）会生成在项目工程根目录

- `.env`（可选）：存储项目环境变量，通常用于存储项目敏感信息

- `.csv`（可选）：项目数据文件，用于进行数据驱动

- `reports`：默认生成测试报告的存储文件夹



## 测试用例结构

- 测试用例集（testsuite）：对应一个文件夹，包含单个或多个测试用例（`YAML/JSON`）文件
- 测试用例（testcase）：对应一个 `YAML/JSON` 文件，包含单个或多个测试步骤
- 测试步骤（teststep）：对应 `YAML/JSON` 文件中的一个 `test`，描述单次接口测试的全部内容，包括发起接口请求、解析响应结果、校验结果等

## 分层模型

![binary](/images/httprunner/jigou.png)

# 实战

一个我司实际用例: 用户登录获取用户token，通过token获取用的信息

## 目录结构如下所示：

```
tests
├── .env
├── data
│   ├── data.csv
├── api
│   ├── get_userinfo.yml
│   ├── mmno_login.yml
├── debugtalk.py
├── testcases
│   ├── get_userinfo.yml
└── testsuites
    ├── user.yml.yml
```

## 环境变量

```properties
USERNAME=autotest
PASSWORD=123456
USER_URL=https://test-user.memeyule.com
AMBER_URL=https://test-amber.memeyule.com
API_URL=https://test-api.memeyule.com
```

## 接口定义

```yaml
#接口定义API   接口描述需要尽量保持完整，做到可以单独运行
#账号和密码登录
name: 获取用户信息
base_url: ${ENV(API_URL)}  #从环境变量获取 .env文件
variables:
    expected_status_code: 200
request:
    url: /user/info/$access_token  #testcase 传递过来的变量
    method: GET
    headers:
        Content-Type: "application/x-www-form-urlencoded"
validate:
    - eq: ["status_code", $expected_status_code]
```

接口描述需要尽量保持完整，做到可以单独运行

## 测试用例

```yaml
# 测试用例(testcase)
# 获取用户信息
config:
    name: "demo testcase"           #测试用例名称
    variables:
        username: ${ENV(USERNAME)}  #环境变量 .env文件
        password: ${ENV(PASSWORD)}
    base_url: "https://test-user.memeyule.com"

teststeps:
-
    name: get_token           # 测试步骤1  获取用户token
    api: api/mmno_login.yml   # 调用登录api
    extract:                  # 获取token 传递的下个接口
        - access_token: content.access_token
    validate:
        - eq: ["status_code", 200]

-
    name: get_userinfo        # 测试步骤2  获取用户信息
    api: api/get_userinfo.yml # 调用获取用户信息api
    variables:
      access_token: $access_token # 将token值传入接口中使用
    validate:
      - eq: ["content.code", 1]
```

## 测试用例集

```yaml
# 测试用例集（testsuite）测试用例集 是 测试用例(testcase)的 无序 集合，集合中的测试用例应该都是相互独立，不存在先后依赖关系的。
config:
    name: "用户相关"
    base_url: "https://test-user.memeyule.com"

testcases:
-
    name: call user test case
    testcase: testcases/get_userinfo.yml   #获取用户信息

```

## 运行

```shell
$ hrun testsuites/
INFO     HttpRunner version: 2.2.5
INFO     Loading environment variables from D:\project\HttpRunner\Samples\demo\.env
INFO     Start to run testcase: call user test case
get_token
INFO     POST https://test-amber.memeyule.com/api/v1/oauth2/token
INFO     status_code: 200, response_time(ms): 143.01 ms, response_length: 123 bytes

.
get_userinfo
INFO     GET https://test-api.memeyule.com/user/info/f2f6ef05e5608c1a4130ceafd76a8849
INFO     status_code: 200, response_time(ms): 196.01 ms, response_length: 387 bytes

.

----------------------------------------------------------------------
Ran 2 tests in 0.343s

OK
INFO     Start to render Html report ...
INFO     Generated Html report: D:\project\HttpRunner\Samples\demo\reports\1565860957.html
```

## reports

![binary](/images/httprunner/report.png)



# 其他工具

## **项目脚手架**

在 `HttpRunner` 中实现了一个脚手架工具，可以快速创建项目的目录结构。

```shell
$ hrun --startproject demo
Start to create new project: demo
CWD: /Users/debugtalk/MyProjects/examples

created folder: demo
created folder: demo/api
created folder: demo/testcases
created folder: demo/testsuites
created folder: demo/reports
created file: demo/debugtalk.py
created file: demo/.env
```

