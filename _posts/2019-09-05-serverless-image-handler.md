---
layout: post
title:  "Serverless Image handler"
toc: true
---
## 1.Introduction

为了帮助客户提供低延迟的网站响应，并降低图像优化、操作和处理的成本，AWS提供了无服务器图像处理程序，该解决方案结合了高度可用、受信任的AWS服务和开源的图像处理套件Sharp，以实现快速响应。在aws云上进行经济高效的图像处理。此参考实现自动部署和配置一个无服务器架构，该架构针对动态图像操作进行了优化，使用amazon simple storage service（amazon s3）以低成本实现可靠和持久的云存储。

## 2.Architecture

aws提供了一个简单的解决方案，可以自动部署和配置为动态图像操作而优化的无服务器体系结构。下图展示了无服务器图像处理程序体系结构，您可以使用该解决方案和aws cloudformation模板在几分钟内进行部署。

![image-20190905134239244](https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/serverless-image-handler/image-20190905134239244.png)



#### 注意:

此解决方案适用于具有公开访问的客户，这些客户希望提供动态更改或操作其公共图像的选项。因此，此模板将在您的帐户中创建一个可公开访问、未经身份验证的S3静态存储网站和Amazon API网关端点，允许任何人访问它。

AWS cloudformation模板部署了Amazon s3存储桶，api网关和aws lambda函数。S3提供了静态网站，通过网站交互式的对图片进行处理。api网关提供端点资源并触发lambda函数。lambda函数从客户自定义的Amazon s3 存储桶中检索图像，并使用sharp将图像的修改版本返回给api网关。

### 准备工作

1. 下载cloudformation模版

   cloudfromation模版下载地址：https://image-handler-cn-public.s3.cn-north-1.amazonaws.com.cn/image-handler-cn.yml

2. 部署上一步的cloudformation模版

3. cloudformation模版部署完成后查看输出内容

   ![image-20190905135416819](https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/serverless-image-handler/image-20190905135416819.png)

   说明：

   api的值即api网关的url地址

   demoui的值即静态网站的url地址

4. 创建存放待处理图片的amazon s3桶

5. 将待处理的图片上传至上述桶中

## 3.how to use

本部分内容介绍如何使用部署好的无服务器架构来处理图片

#### 通过浏览器使用

1. 打开浏览器，输入demoui的地址。如下图

   ![image-20190905140539232](https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/serverless-image-handler/image-20190905140539232.png)

2. 在**图片源**部分，分别输入存放图片的**s3桶名称**和**图片名称**，点击**导入**。如下图

   ![image-20190905144920116](https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/serverless-image-handler/image-20190905144920116.png)

3. 设置图片处理的参数，在**图片编辑**部分修改相应参数。如下图

   ![image-20190905145016474](https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/serverless-image-handler/image-20190905145016474.png)

说明：

**图片预览**下方的**请求正文**是发送给lambda函数进行处理的参数，**格式化URL**是可以直接调用的地址。

#### 通过api调用-1

除了通过浏览器对图片进行处理之外，还以通过直接调用api的方式。

1. 将图片处理的参数组装成json文本

   ```json
   {
     "bucket": "pwmimage",
     "key": "ber.jpg",
     "edits": {
       "resize": {
         "width": 300,
         "height": 300,
         "fit": "cover"
       },
       "negate": true
     }
   }
   ```

   更多处理方式请参考：https://sharp.pixelplumbing.com/en/stable/

2. 将文本转为base64格式

   ```shell
   (base) ➜  ~ echo '{ "bucket": "pwmimage", "key": "ber.jpg", "edits": { "resize": { "width": 300, "height": 300, "fit": "cover" }, "negate": true } }'|base64
   eyAiYnVja2V0IjogInB3bWltYWdlIiwgImtleSI6ICJiZXIuanBnIiwgImVkaXRzIjogeyAicmVzaXplIjogeyAid2lkdGgiOiAzMDAsICJoZWlnaHQiOiAzMDAsICJmaXQiOiAiY292ZXIiIH0sICJuZWdhdGUiOiB0cnVlIH0gfQo=
   ```

3. 拼接api接口调用地址，将api地址拼接上一步中的base64值，如

   ```
   https://j4h1yfm3bi.execute-api.cn-north-1.amazonaws.com.cn/prod/eyAiYnVja2V0IjogInB3bWltYWdlIiwgImtleSI6ICJiZXIuanBnIiwgImVkaXRzIjogeyAicmVzaXplIjogeyAid2lkdGgiOiAzMDAsICJoZWlnaHQiOiAzMDAsICJmaXQiOiAiY292ZXIiIH0sICJuZWdhdGUiOiB0cnVlIH0gfQo=
   ```

4. 通过浏览器测试

   ![image-20190905145155083](https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/serverless-image-handler/image-20190905145155083.png)

#### 通过api调用-2

1. 拼装url地址，拼接格式为：api地址+s3存储桶名称+图片处理参数+图片名称

   比如将图片大小裁剪为300x300，拼接后的url地址为：

   https://j4h1yfm3bi.execute-api.cn-north-1.amazonaws.com.cn/prod/pwmimage/300x300/ber.jpg

   更多处理方式请参考：https://thumbor.readthedocs.io/en/latest/getting_started.html

2. 通过浏览器测试

   ![image-20190905145252698](https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/serverless-image-handler/image-20190905145252698.png)
