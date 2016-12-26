---
title: Auto Build System for Chromium
date: 2016-12-26
---

新学期刚开始没多久便拿到了Intel上海的实习Offer，于是选择了翘课实习，至今已经三多月了。
实习第一个阶段的任务便是开发一个针对Chromium的自动化编译平台。

## 需求 ##
按照最初的需求，新平台包含以下功能:
*   Auto fetch and build Chromium project at regular time.
*   Build commits according to demand.
*   Quick query to a commit and download built binaries.

后来经过我提议，决定搭建一个**分布式编译集群**，采用Master-Slave结构，将组里淘汰下来的旧电脑充分利用，作为build worker加入Machine pool。

## 设计 ##
系统主要分成四个模块：**Master**, **Collector**, **Database**, **Worker**

其中的**Master**，**Collector**，**Database**相当于Master-Slave结构中的Master，而Worker就是Slave,
结构关系如下图所示：

TBD

*   **Master**
通过消息队列与Worker进行交互，提供数据库管理、添加build任务等操作的API。提供Admin Site给用户进行基于Web的可视化管理。
    *   Admin Site: 给用户提供可视化的管理界面
    *   Database API: 封装SQLAlchemy给其他功能提供数据库操作API
    *   Database Keeper: 监听消息队列中的任务状态更新以及查询信息
    *   Scheduler: 定期更新代码库并将新的代码库信息同步进数据库，同时将需要build的commit加入task队列。

*   **Collector**
独立于Master，负责通过HTTP接收Worker build结束之后提交的log和binaries，通过Web提供查询以及下载的API。

*   **Database**
用于存储代码库的Commit以及Branch信息，Task的build情况等，驱动Master。

*   **Worker**
每个Worker维护一个Chromium的code base，通过消息队列获取build task，包括需要build的commit的commit hash或者version No.。
Build的过程中通过消息队列更新当前任务的完成进度以及自身状态。Build结束后将Binaries和Log打包，通过HTTP上传给Collector，并接受下一个任务。

任务分发流程：
*   添加任务：

TBD

## 实现 ##
技术栈：
*   语言：Python3.4(后端)，HTML,CSS,JS(Admin Site前端)
*   Web框架：[Flask](http://flask.pocoo.org/)
*   ORM工具：[SQLAlchemy](http://www.sqlalchemy.org/)
*   消息队列：[RabbitMQ](http://www.rabbitmq.com/)
*   数据库：[MariaDB](https://mariadb.org/)


## 拓展 ##
*   增加邮件提醒功能，使用SMTPLib
*   使用Docker快速自动化部署Worker

## 总结&感想 ##
根据需求，从设计到开发全都是一人完成，很好地锻炼了我系统模块化设计的能力。“全栈”式的开发也让我对整个系统流程有了更深的理解，为接以后的项目和工作打下了坚实基础。
技术发展与迭代的速度很快，不能只拘泥于工具的使用，要深入理解其背后的设计逻辑，才能应得万变。
