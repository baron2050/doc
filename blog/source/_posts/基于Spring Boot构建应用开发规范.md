---
title: 基于Spring Boot构建应用开发规范
---

# 1.规范的意义和作用

- 编码规范可以最大限度的提高团队开发的合作效率
- 编码规范可以尽可能的减少一个软件的维护成本 , 并且几乎没有任何一个软件，在其整个生命周期中，均由最初的开发人员来维护
- 编码规范可以改善软件的可读性，可以让开发人员尽快而彻底地理解新的代码
- 规范性编码还可以让开发人员养成好的编码习惯，甚至锻炼出更加严谨的思维

# 2.代码仓库规范

## 2.1公共组件

- 公共组件通常指Java库，提供特定问题的处理程序包
- 公共组件仓库地址：[ https://git.company.com/java-library-group](https://git.company.com/java-library-group)
- 公共组件的坐标命名规范
  - 分组编号：<groupId>com.company.library</groudId> 固定取值
  - 组件名称：<artifactId>name</artifactId> name根据组件名称定义
  - 组件版本：<version>x.y.z</versio> x.y.z根据组件实际版本情况定义

## 2.2服务组件

- 服务组件通常指可以独立部署，运行，维护的服务程序包
- 服务组件仓库地址：[ https://git.company.com/server-microservice-group](https://git.company.com/server-microservice-group)
- 应用组件的坐标命名规范
  - 分组编号：<groupId>com.company.server</groudId>固定取值
  - 组件名称：<artifactId>name</artifactId> name根据组件名称定义
- 组件版本：<version>x.y.z</versio> x.y.z根据组件实际版本情况定义

# 3开发环境规范

- 开发环境：JDK1.7+
- 开发工具：IntelliJ IDEA 2017（安装Lombok Plugin）
- 构建工具：Maven3.x
- 代码管理工具：Git /TortoiseGit

# 4.项目结构规范

## 4.1简述

一个项目对应代码仓库中的一个仓库，项目结构是指一个基于Maven创建的项目目录结构。公共组件项目，通常会创建一个Maven普通项目。服务组件项目，通常会创建一个Maven聚合项目，并在聚合项目目录下创建多个继承Maven聚合项目的Maven模块，它们一起作为服务组件项目的组成部分。

## 4.2项目名

- 要求
  - 英文名称，作为仓库，项目，项目根目录，组件（公共组件，服务组件）的名称
  - 中文名称，用于代码仓库的描述
  - 项目名称和代码仓库的名称保持一致
- 定义
  - 项目名称通常由团队负责人确定
- 示例
  - 项目中文名：人脸数据仓库
  - 项目英文名：data-warehouse-face
  - 项目目录名：data-warehouse-face
  - 项目仓库地址：[ https://git.company.com/server-microservice-group/data-warehouse-face.git](https://git.company.com/server-microservice-group/data-warehouse-face.git)
  - 初始版本：1.0.0
  - 示例是一个服务组件，根据上面定义的信息确定该服务组件的Maven坐标命名：

```html
<groupId>com.company.server</groupId>
<artifactId>data-warehouse-face</artifactId>
<version>1.0.0</version>
1.2.3.
```

## 4.3模块命名

- 要求
  - 模块名称：{项目名称}-{模块名称} 模块名称简洁体现职责
  - 模块名字作为模块组件的名称
- 示例
- 人脸数据仓库的数据接入模块名称：data-warehouse-face-access

## 4.4项目目录

- 项目目录遵循Maven约定目录格式

## 4.5源码目录

- 源码目录指：{项目目录}/src/main/
- 打包定义目录：src/main/assembly
- 代码目录：src/main/java
- 资源目录：src/main/resources
  - /db：数据库脚本归档
  - /data：内部依赖数据归档
- 文档目录：src/main/docs
- 脚本目录：src/main/bin
  - run-manage.sh 运行管理脚本（通过参数start, stop, status, help info控制程序运行）
  - sh：服务组件启动脚本
  - sh：服务组件停止脚本

# 5.编码规范

## 5.1包规范

- 项目基本包：com.company.{项目英文名（较长时适当简化）}.{模块名（可选）}
- config：配置类
- startup：与服务启动相关的类
- client：提供客户端实现的相关类
- common：公共类，定义常量类，组件
- entity: 数据库相关的实体类
- model:数据模型类(参数模型，数据传输模型等）
- control:控制层接口
- service: 服务层
- dao：数据库访问层

## 5.2日志记录

- 统一使用SLF4j接口

## 5.3异常处理

- 运行时异常：通过参数检查等方式避免或抛出运行时异常，日志记录
- 检查异常：检查异常需要捕获，处理，日志记录

## 5.4接口定义

- 原则
  - 接口地址定义表明用意
  - 接口地址定义清晰，简洁，无歧义
  - 同一个服务组件的接口定义具有一致性
- 格式
  - 控制类的顶层地址格式:/{顶层分类名}，例如：/library 人员库相关接口的顶层地址
  - 接口定义使用Swagger的API注解说明
  - 标注完整的请求信息，请求方法，请求地址，参数可选性，接口描述
- 请求方式
  - GET URL-Params
  - POST Form-Data
  - POST RequestBody(JSON格式)
  - POST Mulitpart
- 响应方式
  - 统一的响应模型

## 5.5辅助工具

- 字符串处理：apache common-lang3
- 时间日期处理：joda-time
- JSON处理：Gson，Fastjson
- 集合扩展工具：guava
- 文件和流处理：commons-io
- 编解码：commons-codec
- 建议：尽可能使用开源组件

## 5.6代码注释

- 类，接口，枚举顶层注释
- 接口方法注释
- 静态方法注释
- 公开方法注释
- 类的属性字段注释
- 常量注释
- 不限于以上

# 6.代码控制规范

## 6.1拉取原则

- 强制
  - 每日开始工作拉取
- 约定
  - 提交之前拉取

## 6.2提交原则

- 强制
  - 提交代码必须构建成功（比如：编译，打包成共）
  - 提交代码必须完整（比如：少提文件）
  - 提交代码必须忽略到本地临时文件（比如：target, logs, .idea, *.iml,dist 等）
- 约定
  - 完成一个功能提交
  - 修改一个Bug修改提交
  - 解决冲突提交
  - 每日结束工作提交

## 6.3提交注释

- 强制
  - 中文填写注释
  - 注释反映本次提交变更情况
- 约定
  - 注释描述添加前缀，前缀如下
    - [创建] 通常在项目创建时使用
    - [新增]
    - [修改]
    - [删除]
    - [修复-number] 修复Bug使用，number是Bug编号

# 7.构建规范

## 7.1公共组件构建规范

- 构建输出组件包
- 构建输出组件源码包
- 构建发布到公司私有仓库

## 7.2服务组件构建规范

- 服务组件包命名：{组件名称}-{版本号}-bin.zip
- 构建输出到工程根目录下的dist/{组件名称}-{yyyyMMddHH}目录