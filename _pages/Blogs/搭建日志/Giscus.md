---
title: "Example Post: no thumbnail image"
date: "2023-12-01"
---

# Giscus部署指南

前置步骤：已完成公开github-pages项目的基础搭建

##### 第一步：GitHub为指定项目安装giscus应用

​	[GitHub Apps - giscus](https://github.com/apps/giscus)在该网页登录自己的GitHub账号后即可安装，可选择为所有项目或者指定项目安装

##### 第二步：为项目开启DIsscussions

​	来到项目的Settings

​						--General

​						----Features

​						------Disscusions	

![Discussion](https://i.ibb.co/P1FV02D/giscus-00.png)

勾选开启即可

##### 第三步：来到[官网](https://giscus.app/zh-CN#repository)启用giscus

​	在官网此处![image-20240425193337698](C:\Users\Wang\Desktop\wk\blcm996.github.io\assets\img\image-20240425193337698.png)

输入自己的Github-pages公开项目地址，在下方选择

![image-20240425193554333](C:\Users\Wang\Desktop\wk\blcm996.github.io\assets\img\image-20240425193554333.png)

即可在下方”启用giscus“处找到用于嵌入页面的脚本代码

![123](https://i.ibb.co/Z154x8P/giscus-04.png)

最后将对应的内容补充到根目录的_config.yml中即可

```yml
# External API
giscus_repo: "[ENTER REPO HERE]"
giscus_repoId: "[ENTER REPO ID HERE]"
giscus_category: "[ENTER CATEGORY NAME HERE]"
giscus_categoryId: "[ENTER CATEGORY ID HERE]"
```

推荐博客：

[Hugo博客使用giscus作为评论系统 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/569436383)
