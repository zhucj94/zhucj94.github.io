---
title: activiti
date: 2020-01-15 17:43:01
tags: [activiti,工作]
declare: true
---
### 与SpringBoot整合
+ yml中加入如下配置
```
spring:
  activiti:
    check-process-definitions: true #自动检查、部署流程定义文件
    database-schema-update: true #自动更新数据库结构
    process-definition-location-prefix: classpath:/processes/ #流程定义文件存放目录
    db-history-used: true #启用历史,默认为false,不启用将不会创建历史表
    history-level: full #历史级别
```
+ 在resource下创建processes文件加，与process-definition-location-prefix对应
+ 某些版本，启动类需要排除SecurityAutoConfiguration
```
@SpringBootApplication(exclude = SecurityAutoConfiguration.class)
```
+ 启动项目后会直接在数据创建25张表，并且自动部署bpmn
<!-- more -->

### 表说明
+ 搜索
+ ACT_GE_* (GE->General)通用数据表
+ ACT_RE_* (RE->Repository)流程定义存储表
+ ACT_ID_* (ID->Identity)身份信息表
+ ACT_RU_* (RU->Runtime)运行时数据表
+ ACT_HI_* (HI->History)历史表
