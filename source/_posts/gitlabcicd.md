---
title: 如何配置gitlab cicd文件
date: 2022/7/27 11:28:00   
tags: 
- devops
- 流水线
categories: 
- 运维
---


## Gitlab Runner

在项目根目录下添加 .gitlab-ci.yml 文件，整个持续集成系统是 Gitlab 自带的，要做的就是添加一个 Runner 到系统里来解析文件中的 script 部分。

下面是如何编写.gitlab-ci.yml文件的例子

## example

```
stage:
  - build
  - image
  - deploy

Build:
  stage: build
  only:
    - main
  script:
    

```
## ci文件

### stage 
从7.12版本开始，GitLab CI使用YAML文件(.gitlab-ci.yml)来管理项目配置。该文件存放于项目仓库的根目录，它定义该项目如何构建。

开始构建之前YAML文件定义了一系列带有约束说明的任务。这些任务都是以任务名开始并且至少要包含script部分

script可以直接执行系统命令(例如：./configure;make;make install)或者是直接执行脚本(test.sh)。

任务是由Runners接管并且由服务器中runner执行。更重要的是，每一个任务的执行过程都是独立运行的。

```
stages:
 - build
 - test
 - deploy
```
首先，所有build的jobs都是并行执行的。

所有build的jobs执行成功后，test的jobs才会开始并行执行
。
所有test的jobs执行成功，deploy的jobs才会开始并行执行。

所有的deploy的jobs执行成功，commit才会标记为success

任何一个前置的jobs失败了，commit会标记为failed并且下一个stages的jobs都不会执行。


### script
before_script:  定义在每个job之前运行的命令

after_script: 定义在每个job之后运行的命令

### variables
GItLab CI 允许在.gitlab-ci.yml文件中添加变量，并在job环境中起作用。因为这些配置是存储在git仓库中，所以最好是存储项目的非敏感配置，例如：
```
variables:
  DATABASE_URL:"postgres://postgres@postgres/my_database"
```
这些变量可以被后续的命令和脚本使用。服务容器也可以使用YAML中定义的变量，因此我们可以很好的调控服务容器。变量也可以定义成job level。

除了用户自定义的变量外，Runner也可以定义它自己的变量。CI_COMMIT_REG_NAME就是一个很好的例子，它的值表示用于构建项目的分支或tag名称。除了在.gitlab-ci.yml中设置变量外，还有可以通过GitLab的界面上设置私有变量。 
  
- 设置groups variables

```
Key: Must be one line, with no spaces, using only letters, numbers, or _.
Value: No limitations.
Type: File or Variable.
Environment scope Optional. All, or specific environments. 
Protect variable Optional. If selected, the variable is only available in pipelines that run on protected branches or tags.
Mask variable Optional. If selected, the variable’s Value is masked in job logs. The variable fails to save if the value does not meet the masking requirements.

```

- gitlab-ci.yml

```
variables:
  TEST_VAR: "All jobs can use this variable's value"

job1:
  variables:
    TEST_VAR_JOB: "Only job1 can use this variable's value"
  script:
    - echo "$TEST_VAR" and "$TEST_VAR_JOB"
```

- project variables
  

### cache


## Jobs
.gitlab-ci.yml允许指定无限量jobs。每个jobs必须有一个唯一的名字，而且不能是上面提到的关键字。job由一列参数来定义jobs的行为。
```
job_name:
  script:
    - rake spec
    - coverage
  stage: test
  only:
    - master
  except:
    - develop
  tags:
    - ruby
    - postgres
  allow_failure: true
```
keyword requirements description
 
script	yes	Runner执行的命令或脚本

image	no	所使用的docker镜像，查阅使用docker镜像

services	no	所使用的docker服务，查阅使用docker镜像

stage	no	定义job stage（默认：test）

variables	no	定义job级别的变量

only	no	定义一列git分支，并为其创建job

except	no	定义一列git分支，不创建job

tags	no	定义一列tags，用来指定选择哪个Runner（同时Runner也要设置tags）

allow_failure	no	允许job失败。失败的job不影响commit状态

when	no	定义何时开始job。可以是on_success，on_failure，always或者manual

dependencies	no	定义job依赖关系，这样他们就可以互相传递artifacts
cache	no	定义应在后续运行之间缓存的文件列表

before_script	no	重写一组在作业前执行的命令

after_script	no	重写一组在作业后执行的命令

environment	no	定义此作业完成部署的环境名称

coverage  no  定义给定作业的代码覆盖率设置

## artifacts

artifacts用于指定成功后应附加到job的文件和目录的列表。只能使用项目工作间内的文件或目录路径。如果想要在不通的job之间传递artifacts，请查阅依赖关系。以下是一些例子：

发送binaries和.config中的所有文件：
```
artifacts:
  paths:
  - binaries/
  - .config
```


### includes

Possible inputs: The include subkeys:

- include:local
- include:file
- include:remote
- include:template

#### inclue:file
To include files from another private project on the same GitLab instance, use include:file. You can use include:file in combination with include:project only.

Example of include:file:
```
include:
  - project: 'my-group/my-project'
    file: '/templates/.gitlab-ci-template.yml'
```