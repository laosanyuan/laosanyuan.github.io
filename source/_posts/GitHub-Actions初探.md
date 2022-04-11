---
title: GitHub Actions初探
date: 2021-10-04 01:29:10
tags:
  - GitHub
---

## CICD的概念

CICD是持续集成（Continuous Integration）和持续部署（Continuous Deployment）的简称。是指在开发过程中自动执行一系列脚本来减少开发过程中引入的bug，在新代码从开发到部署过程中尽量减少人工的介入。

持续集成是指频繁多次地将代码集成到主干。在远程仓库push代码后，在这次提交合并至主分支前预先进行一系列测试、构建等流程。对于每次提交的代码，都进行一次脚本集成操作，保持质量的同时，降低了引入错误的概率。持续部署是在持续集成的基础上更进一步，指将仓库至指定分支部署至产品环境。

## GitHub Actions 

在GitHub上的Actions可以非常方便的支持项目总的持续集成和部署需求。在每个仓库的Tab菜单下都可以找到对应的Actions页面，并且提供免费的集成服务。用户可以通过为项目配置简单的Actions文件达到使用CICD的目的。本质上是提供了一个可以指定配置的服务器，所以Action所能支持的玩法又不仅限于CICD。

持续集成由多种操作组成，每个项目的需求有相同的需要也有特殊的情况，所以GitHub将持续集成以Action作为单元拆分。每个Action作为独立的脚本文件，存放到代码仓库，供自己甚至他人使用。整个集成的过程就变成了一系列单独Action的组合，也就是所谓的Actions，达到了最终完整的集成目的。

由于GitHub对Action的支持，用户就可以轻松的使用现成的Action进行使用。   

## Actions的基础用法

### Actions的一些基本概念

* workflow：持续集成运行一次的过程，是工作流。   

* events：仓库触发运行工作流的具体行为，具体支持的事件种类可以查看[GitHub官方支持列表](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows)。

* 组成workflow的工作任务部分，一个workflow中可包含多个job。   

* step：job的组成部分。   

* action：每个step可执行一个或多个action（命令），是具体的操作或行为。

workflow文件存放于代码仓库的`.github/workflows`目录下，文件采用YAML格式，名称任意，但需以`.yml`结尾。一个库中可以包含多个workflow，分别存放在对应的文件个体中。   

### workflow基本字段

* name：workflow名称，根据需求自行设置。   
* on：触发workflow执行的条件event，如push。on事件可以以数组形式存在，如on: [push, pull_request]，指定触发事件时可以限定分支或标签。
* jobs：workflow的工作主体。jobs下可以列出需要执行的每一项job，名称自己定义。
  * jobs.<job_id>.needs：用于指定job之间的以来顺序，可以通过配置来指定当前的任务关系。选填。   
  * jobs.<job_id>.runs-on：指定运行所需要的虚拟机环境，是必填字段。   
  * jobs.<job_id>.env：环境变量   
  * jobs.<job_id>.steps：指job的运行步骤，每个steps下可包含一个或多个step。   
    * jobs<job_id>.steps.run：该步骤运行的命令或者action。   
    * jobs<job_id>.steps.env：该步骤所需的环境变量。   
    * jobs<job_id>.steps.uses：指定在当前step中需要运行的action，可以在此处指定使用他人或自己写好的Action。   
    * jobs<job_id>.steps.with：action需要的参数。   

## 举例

研究了Actions之后，在我的[小项目](https://github.com/laosanyuan/HuoHuan)上验证了一下，workflow文件内容如下：

```yaml
# workflow名称
name: Huohuan - CI

# 触发条件 - master分支下push代码触发
on:
  push:
    branches: [ master]

# Actions任务Jobs
jobs:
  # job，案自己需求命名
  build:
  # 策略，具体用法可参考[strategy context](https://docs.github.com/en/actions/learn-github-actions/contexts#strategy-context)
    strategy:
      matrix:
        configuration: [Debug, Release]
    # 运行环境
    runs-on: windows-latest

    steps:
    # 拉取代码，`actions/checkout@v3`表示调用账户actions下checkout项目的v3分支
    - name: Checkout
      uses: actions/checkout@v3
      
    - name: 安装.Net Core
      uses: actions/setup-dotnet@v1
      with:
        dotnet-version: 6.0.x
        
    - name: 安装Nuget
      uses: nuget/setup-nuget@v1
    - run: nuget restore src\HuoHuan.sln
    
    - name: 编译
      run: dotnet build src\HuoHuan.sln --no-restore -c ${{ matrix.configuration }}
      
      # ${{}}可以使用上下文参数。
    - name: 单元测试
      run: dotnet test src\HuoHuan.sln -c $env:Configuration
      env:
        Configuration: ${{ matrix.configuration }}
```

更多用法可以参考[官方文档](https://docs.github.com/en/actions)。