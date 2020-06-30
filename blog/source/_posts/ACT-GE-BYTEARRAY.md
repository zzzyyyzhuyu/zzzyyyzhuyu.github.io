---
title: ACT_GE_BYTEARRAY 二进制数据流表
date: 2020-06-30 18:33:13
categories:
     - 工作流相关
tags: 
     - activity6
---

# ACT_GE_BYTEARRAY 二进制数据流表

#### 说明
保存流程定义图片和xml、Serializable(序列化)的变量,即保存所有二进制数据

#### 字段列表

| 字段名称  | 字段描述 | 数据类型 | 取值说明  |
 | :---    | :---    | :--- |  :---   |
 |ID_	   |ID_	|nvarchar(64)  |	主键ID |
 |REV_	   |乐观锁	|int		|	Version(版本)|
 |NAME_	   |名称	|nvarchar(255)	| 部署的文件名称，如：leave.bpmn.png,leave.bpmn20.xml|
 |
 |DEPLOYMENT_ID_	 |部署ID	|nvarchar(64)	|部署表ID|
 |BYTES_   |字节  |	varbinary(max)	| 部署文件 |
 |GENERATED_  |是否是引擎生成 |	tinyint	|	0为用户生成，1为activiti生成 |
