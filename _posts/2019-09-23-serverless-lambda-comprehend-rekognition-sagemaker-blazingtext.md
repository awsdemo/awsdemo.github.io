---
layout: post
title:  "基于文本的商品分类识别及图片审核方案"
toc: true
---



## 概述

电商网站通常有大量的商品信息，通常包括两类信息文本（商品名称，描述）和图片信息。根据某些国家及地区的法律及法规要求，成人用品在某些区域销售属于禁售品类，第三发卖家有时通过删除关键字的方法绕过网站基于关键字匹配禁售机制，导致法律风险。本文描述了基于AWS Comprehend、SageMaker自带的内建BlazingText算法及Facebook Fasttext算法和AWS ECS Fargate自建方案来实现通过对商品文本的基于NLP（自然语言处理）的品类自动判别，从而不依赖个别关键字。同时使用AWS的Rekognition服务来轻松实现对含NSFW（Not Safe for Work）内容的商品图片的审核。



## 架构总览

![]()

![Tite goes here](https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/aws-NLP/Peggie.jpeg)

## 实例代码

```python
import boto3
```

