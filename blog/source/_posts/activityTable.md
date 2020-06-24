---
title: activity相关表结构介绍
date: 2020-06-24 15:06:48
categories:
     - 工作流相关
tags: 
     - activity6
---

# 表结构说明
#### 命名规则   
- Activiti 使用到的表都是 ACT_ 开头的。表名的第二部分用两个字母表明表的用途。
- ACT_GE_ （GE） 表示 general 全局通用数据及设置，各种情况都使用的数据。
- ACT_HI_ （HI） 表示 history 历史数据表，包含着程执行的历史相关数据，如结束的流程实例，变量，任务，等等
- ACT_ID_ （ID） 表示 identity 组织机构，用户记录，流程中使用到的用户和组。这些表包含标识的信息，如用户，用户组，等等。
- ACT_RE_ （RE） 表示 repository 存储，包含的是静态信息，如，流程定义，流程的资源（图片，规则等）。
- ACT_RU_ （RU） 表示 runtime 运行时，运行时的流程变量，用户任务，变量，职责（job）等运行时的数据。Activiti 只存储实例执行期间的运行时数据，当流程实例结束时，将删除这些记录。这就保证了这些运行时的表小且快。

#### 表结构说明

###### 一般数据(ACT_GE_)
   
 | 表名  | 说明 |
 | :--- | :---|
 |ACT_GE_BYTEARRAY|	二进制数据表，存储通用的流程定义和流程资源|
 |ACT_GE_PROPERTY|系统相关属性，属性数据表存储整个流程引擎级别<br>的数据，初始化表结构时，会默认插入三条记录|
 
###### 流程历史记录(ACI_HI_) 

 |  表名  |  说明  |
 | :--- | :---|
 | ACT_HI_ACTINST	 | 历史节点表
 | ACT_HI_ATTACHMENT |	历史附件表
 | ACT_HI_COMMENT	 | 历史意见表
 | ACT_HI_DETAIL	|  历史详情表，提供历史变量的查询
 | ACT_HI_IDENTITYLINK |	历史流程人员表
 | ACT_HI_PROCINST	| 历史流程实例表
 | ACT_HI_TASKINST	| 历史任务实例表
 | ACT_HI_VARINST	| 历史变量表 
 
###### 用户用户组表(ACT_ID_ )

 |  表名  |  说明  |
 | :--- | :---|
 | ACT_ID_GROUP |	用户组信息表
 | ACT_ID_INFO	| 用户扩展信息表
 | ACT_ID_MEMBERSHIP |	用户与用户组对应信息表
 | ACT_ID_USER	| 用户信息表
  
###### 流程定义表(ACT_RE_)

 |  表名  |  说明  |
 | :--- | :---|
 | ACT_RE_DEPLOYMENT |	部署信息表
 | ACT_RE_MODEL	 | 流程设计模型部署表
 | ACT_RE_PROCDEF |	流程定义数据表 
 
###### 运行实例表 (ACT_RU_) 

 |  表名  |  说明  |
 | :--- | :---|
 | ACT_RU_EVENT_SUBSCR | 运行时事件 throwEvent、catchEvent 时间监听信息表
 | ACT_RU_EXECUTION | 运行时流程执行实例
 | ACT_RU_IDENTITYLINK | 运行时流程人员表，主要存储任务节点与参与者的相关信息
 | ACT_RU_JOB |	运行时定时任务数据表
 | ACT_RU_TASK | 运行时任务节点表
 | ACT_RU_VARIABLE | 运行时流程变量数据表
 
 ###### 其他表
 |  表名  |  说明  |
 | :--- | :---|
 | ACT_EVT_LOG | 事件日志
 | ACT_PROCDEF_INFO |流程定义的动态变更信息