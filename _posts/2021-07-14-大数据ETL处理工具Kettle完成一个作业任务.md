---
layout:     post
title:      大数据 ETL 处理工具 Kettle 完成一个作业任务
subtitle:   
date:       2021-07-14
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: BigData
tags:
    - BigData
    - 数据仓库
---

### 作业流程

#### 什么是作业流程

简单一句话，作业流程，即是对转换流程进行调度，也可以嵌套转换流程和作业流程。

> 在第二篇中介绍了一些核心概念，已经知道什么是转换等概念，第一篇中通过一个 HelloWorld 级别的实践，快速体验了一把「转换」的流程。

一个作业流程必须包含「START」 组件，可以没有「成功」组件，作业流程中可以嵌套转换流程和作业流程，如下图：

![作业流程](https://gitee.com/yzhw/img/raw/master/img/作业流程.png)

除了调度转换流程还可以做一些其他的工作，比如文件管理（创建一个目录、创建文件、删除一个文件、复制文件等）、条件判断（检查目录是否为空、检查一个文件是否存在等）、脚本（JavaScript、Shell、SQL）执行、发送邮件等等，如下图：

![image-20210713165038894](https://gitee.com/yzhw/img/raw/master/img/image-20210713165038894.png)

#### 新建作业流程

##### 场景

将第一篇中的转换「数据从 CSV 文件复制到 Excel 文件」配置到作业流程中进行执行。

##### 新建

新建作业流程与转换流程类似，快捷键是 `Command + Option + N`。核心组件在 `通用` 分类下，分别将「Start」、「转换」、「成功」拖拽到右侧工作区，按住 shift + 鼠标左键可以建立步骤间的连接，如下图：

![image-20210713171142834](https://gitee.com/yzhw/img/raw/master/img/image-20210713171142834.png)

> 小结：
>
> 与转换流程不同的是，除了步骤之间有 `连接状态`（箭头颜色深浅），还有 `连接条件`（箭头上的图标，一共三种)。上图的这个作业中包含了所有连接条件：
>
> - 小锁图标，表示不管上一步骤执行结果如何，都执行下一个步骤
> - 红叉图标，表示只有上一步骤执行出错或者返回FALSE，才执行下一步骤
> - 绿勾图标，表示只有上一步骤执行成功或者返回TRUE，才执行下一步骤
>
> 单击连接条件图标可以调整连接条件，`START` 步骤与下一步骤之间的连接条件不可修改

##### 配置

双击「转换」，可以设置作业项名称（推荐设置一个见名知意的名称），点击「浏览」选择转换路径，其他先保持默认，如下图：

![image-20210713171550083](https://gitee.com/yzhw/img/raw/master/img/image-20210713171550083.png)

##### 执行

点击「执行」按钮，开始执行作业，并输出日志信息，如下图：

![image-20210713171826845](https://gitee.com/yzhw/img/raw/master/img/image-20210713171826845.png)



### 定时调度

- 「START」组件标识着工作流的开始，也是配置定时任务的地方，右键「START」组件选择「编辑作业入口」

![image-20210713173845856](https://gitee.com/yzhw/img/raw/master/img/image-20210713173845856.png)

> 需要一直保持Spoon处于启动状态，一旦Spoon窗口被误关闭，定时任务就无效了，所以一般不使用 Kettle 自带的这个调度器

- crontab 进行调度，在实际工作中使用了这种方式，示例如下：

```shell
25 4,23 * * 1-6 sh /Users/Yezhiwei/Documents/apps/data-integration/kitchen.sh -file=/usr/schedule/JobFiles/main.kjb --level=minimal>>/usr/schedule/JobFiles/main/1.log
```

### 结语

最后制作一个小视频，完成作业整个流程。

<video id="video" controls="" preload="none">
    <source id="mp4" src="https://gitee.com/yzhw/img/blob/master/img/output(compress-video-online.com).mp4" type="video/mp4">
</video>

