[TOC]

***

#### 创建定时任务

* Android中的定时任务可以有2种实现，一种是javaAPI提供的Timer类，另一种是使用Android的Alarm机制。但是timer类有一个短板，由于Android手机会有自己的休眠策略，当手机长时间不使用时，会让CPU进入