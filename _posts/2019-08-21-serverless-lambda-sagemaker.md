---
layout: post
title:  "Serverless Integrated AI "
toc: true
---

## 1. 实验目的

本实验用于帮助我们在AWS上实现无服务器架构，lambda可以和Dynamo DB，kinesis，S3等多种服务进行配合使用。在本篇文章中，我们将会对Lambda进行一个简单的介绍，然后进行动手实验，将lambda与API Gateway及sagemaker服务结合起来，构建一个cancer prediction系统。

## 2.AWS Lambda

### 2.1 lambda简介

lambda 是一项计算服务，可以使您无需配置或管理服务器即可运行代码。从本质上讲，lambda是一项事件驱动的服务，驱动事件可以是S3，SNS，Kinesis等。lambda需要的计算能力可以按需进行持续扩展，真正实现只为计算资源付费。

<a data-fancybox="gallery" href="https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/lambda1-api.png">
![lambda](https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/lambda1-api.png)</a>

### 2.2 lambda并发问题

#### 以 kinesis+lambda 为例：

​    我们都知道，kineses可以有多个shards(活动分区)，如果kinesis现在有5个shards，那么lambda就会有5个函数进行调用，并做并发的执行。当kinesis的shards数量自动扩展时，lambda也会进行自动扩展



<a data-fancybox="gallery" href="https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/lambdabingfa.png">
![lambda](https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/lambdabingfa.png)</a>



#### 对于流量激增问题：

当前端流量比较小时，lambda会自动分配一些资源。当前端流量变大时，lambda会根据增加的资源进行底层资源的动态扩展，其并发执行函数可以瞬间飙升到3000，之后如果还需扩展会以500为单位继续扩展，直到达到账户的安全限制或并发执行函数足以处理负载。。

 <a data-fancybox="gallery" href="https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/lambdaflow.png">
![lambda](https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/lambdaflow.png)</a>



## 3. Lambda 集成API Gateway和sagemaker



API Gateway用于创建，维护任意规模的REST（无状态）和websocket（有状态） API。使用lambda和api gw可以构建aws无服务器基础设施中 面向应用程序的部分。在本次动手实验中，我们将使用lambda和API GW，并结合sagemaker，构建一个cancer prediction系统。

系统架构图如下：

![lambda](https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/lambdaarchitecture.png)

### 3.1 构建sagemaker模型

打开控制台，搜索sagemaker，创建一个新的笔记本实例。创建完成后，点击 open Jupyter，即可以开始使用sagemaker服务。我们可以选择使用sagemaker内置算法，也可以自己编写训练模型。

 <a data-fancybox="gallery" href="https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/sagemaker.png">
![lambda](https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/sagemaker.png)</a>

此处，为了快速搭建整套系统，采用sagemaker Examples，并选择Breast Cancer Prediction,点击Use，Create Copy。您可以选择自己一步步运行本模型的搭建，也可以在打开的jupyter笔记本中点击cell，run all，一键运行代码。

<a data-fancybox="gallery" href="https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/sagemaker1.png">
![lambda](https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/sagemaker1.png)</a>

代码运行完毕之后，打开sagemaker控制台，点击终端节点，我们可以看到已经部署完成的终端节点。

<a data-fancybox="gallery" href="https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/sagemaker2.png">
![lambda](https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/sagemaker2.png)</a>

由于sagemaker的使用不是本篇文章的重点讲解范围，若在搭建模型的过程中出现问题请参考：[sagemaker入门](https://docs.aws.amazon.com/zh_cn/sagemaker/latest/dg/gs.html)

### 3.2 lambda函数进行sagemaker模型调用

使用sagemaker训练模型的目的就在于能够将模型进行实际应用，所以在模型终端节点部署完成之后我们使用lambda对模型的端点进行调用。首先，在lambda控制台新建一个lambda函数，并赋予执行角色sagemaker的访问权限。运行语言选择python3.6。创建完成之后将下列代码复制到lambda_function中

```python
#给定一个sample，判断乳腺癌患者是隐性还是阳性
import os
import io
import boto3
import json
import csv

# grab environment variables
runtime= boto3.client('runtime.sagemaker')

def lambda_handler(event, context):
    print("Received event: " + json.dumps(event, indent=2))
   
    data = json.loads(json.dumps(event))
    payload = data['data']
    print(payload)
    
    response = runtime.invoke_endpoint(EndpointName="DEMO-linear-endpoint-201908190751",
    ContentType='text/csv',
    Body=payload)
    print(response)
    result = json.loads(response['Body'].read().decode())
    print(result)
    pred = int((result['predictions'][0]['score']>0.5))
    predicted_label = 'M' if pred == 1 else 'B'

		return predicted_label
```

### 3.3 创建API Gateway

在API Gateway控制台，新建一个API，选择REST协议，并为API命名。

<a data-fancybox="gallery" href="https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/apigw.png">
![lambda](https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/apigw.png)</a>

然后，点击操作，创建方法，选择GET，并关联刚才创建的lambda函数，点击保存，在弹出来的提示框中点击确定，为API GW赋权。

<a data-fancybox="gallery" href="https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/apigwget.png">
![lambda](https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/apigwget.png)</a>

接着点击GET方法的集成请求，为其添加执行角色，确保添加的角色具有lambda的访问权限。

<a data-fancybox="gallery" href="https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/apigw2.png">
![lambda](https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/apigw2.png)</a>

API GW创建完成之后，点击操作，部署API.选择部署阶段为[新阶段]，命名阶段并完成部署。

<a data-fancybox="gallery" href="https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/apigw3.png">
![lambda](https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/apigw3.png)</a>

至此，整个无服务器架构搭建完成。

### 3.4 架构测试

打开postman，选择GET请求，并将API gateway测试阶段的调用URL复制过来。将下面代码块中的cancer 测试数据复制到body中，进行架构测试。

<a data-fancybox="gallery" href="https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/test.png">
![lambda](https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/test.png)</a>

```
{"data":"13.49,22.3,86.91,561.0,0.08752,0.07697999999999999,0.047510000000000004,0.033839999999999995,0.1809,0.057179999999999995,0.2338,1.3530000000000002,1.735,20.2,0.004455,0.013819999999999999,0.02095,0.01184,0.01641,0.001956,15.15,31.82,99.0,698.8,0.1162,0.1711,0.2282,0.1282,0.2871,0.06917000000000001"}
```

