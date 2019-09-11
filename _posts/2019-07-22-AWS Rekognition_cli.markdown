---
layout: post
title:  "Rekognition功能展示"
toc: true
---


## 1.概述  
  利用 Amazon Rekognition，可将图像和视频分析轻松添加到您的应用程序。您只需向 Rekognition API 提供图像或视频，然后此服务就能识别物体、人员、文字、场景和活动。它还可以检测到任何不合适的内容。Amazon Rekognition 还提供了高度准确的人脸分析和人脸识别。您可以检测、分析和比较各种使用案例中的人脸，包括用户确认、编录、人员计数和公共安全。Amazon Rekognition 基于同样由 Amazon 计算机视觉科学家开发的成熟且高度可扩展的深度学习技术，每天能够分析数十亿图像和视频- 无需使用任何机器学习专业技能。Amazon Rekognition 包含一个简单易用的 API，该 API 可快速分析存储在 Amazon S3 中的任何图像或视频文件。Amazon Rekognition 始终从新数据进行学习，我们会不断向此服务添加新的标签和人脸识别功能。

## 2.Demo功能

这是一个Amazon Rekognition服务的示例项目。在本演示项目中，我们将展示AWS图像识别技术分别应用于对象场景检测，图像审核，面部分析，面孔比较以及面部识别等方面的实际效果。您可以通过页面演示来体验图像识别技术的准确性与效果。
本演示仅用于试验和参考用途，如果您想开发自己的图像识别应用，请登录AWS官网查看详细信息。  
demo示例：[demo link](http://rekognition111demo.s3-website-ap-southeast-1.amazonaws.com/)  


## 3.如何使用本Demo
开始使用本demo，必须进行以下配置
### 3.1 cognito身份池创建步骤：
在aws cognito中*创建新的身份池*，并允许未授权的身份验证 
```
#create identity-pool
$ aws cognito-identity create-identity-pool --identity-pool-name MyIdentityPool --allow-unauthenticated-identities
#list identity pool，and copy your identity-pool-id
$ aws cognito-identity list-identity-pools --max-results 20
```
配置Cognito用户池所对应的未授权IAM角色具有至少Amazon Rekognition只读操作权限（可以使用已存在的IAM Role），并将该其设置为身份池角色
```
#set-identity-pool-roles
$ aws cognito-identity set-identity-pool-roles --identity-pool-id "YOUR_POOL_ID" \
  --roles authenticated="YOUR_AUTH_ROLE_ARN" \
  --roles unauthenticated="YOUR_UNAUTH_ROLE_ARN" 
```

### 3.2 项目部署：

将整体项目文件克隆至本地：   
```
git clone https://github.com/finishy1995/amazon-rekognition-demo
```
修改js文件夹下的common.js文件，其中：  
常量BUCKET_NAME，替换为项目S3存储桶名  
常量POOL_ID，替换为上述步骤配置的Cognito用户池ID  
其余常量按照实际情况酌情修改
将修改好的项目文件全部上传至S3项目存储桶中。
```
#upload directory amazon-rekognition-demo to s3 bucket
$ aws s3 sync ./amazon-rekognition-demo s3://amazon-rekognition-demo/ 
```

### 3.3 s3 bucket设置：
对新创建的s3 bucket做如下设置：  
#### （1）静态网站设置  
  将所创建的存储桶，进行*静态网站托管*，设置主页为index.html,错误页面为error.html
  
```
#create a new bucket:amazon-rekognition-demo 
$ aws s3 mb s3://amazon-rekognition-demo

#confugure s3 bucket as static website
$ aws s3 website s3://amazon-rekognition-demo/ --index-document index.html --error-document error.html
```

#### （2）权限设置所有人可读 
 
```
# grant read to everyone
$ aws s3api put-bucket-acl --bucket amazon-rekognition-demo --grant-read uri=http://acs.amazonaws.com/groups/global/AllUsers
```

#### （3）存储桶策略设置 
 
```
# create a bucket-policy.json file 
$ vim bucket-policy.json
# insert following json content to bucket-policy.json
$ {
   "Statement": [
      {
         "Effect": "Allow",
         "Principal": "*",
         "Action": "s3:GetObject",
         "Resource": "arn:aws:s3:::amazon-rekognition-demo/*"
      }
  ]
}

#configure bucket policy 
$  aws s3api put-bucket-policy --bucket amazon-rekognition-demo --policy file://bucket-policy.json
``` 
 
#### （4）CORS配置网站地址获得GET, PUT, POST, DELETE操作权限  

```
# create a cors file 
$ vim cors.json
# insert following json content to cors.json
$ {
  "CORSRules": [
    {
      "AllowedOrigins": ["*"],
      "AllowedHeaders": ["*"],
      "AllowedMethods": [ "GET" ,"PUT", "POST", "DELETE"]
    }
  ]
}

#configure CORS 
$ aws s3api put-bucket-cors --bucket amazon-rekognition-demo --cors-configuration file://cors.json
```  
至此，可以点击s3静态托管网站的url,体验rekognition的图像识别功能。若要使用rekognition的面部识别功能，还需要执行以下步骤。
## 4.人脸识别

在面部识别功能的正常使用之前，需要先创建人脸集合，将人脸存储在该集合中。然后该功能会向您提供搜索集合中与要识别的图像匹配度最高的人脸。 

  
参考：[create-collection](https://docs.aws.amazon.com/zh_cn/rekognition/latest/dg/create-collection-procedure.html)    

