---
layout: post
title:  "AWS服务列表"
toc: true
---


## 1.简介
  一套网站应用,用于展示 AWS 区域对应的服务信息，并提供搜索功能以支持区域选型和区域对比。由Serverless架构实现，使用Amazon CloudWatch Events、AWS Lambda、Amazon DynamoDB、Amazon S3、Amazon CloudFront等服务生成并展示数据，前端使用了React、Webpack以及Antd。
## 2.Demo功能 
* 数据每12小时自动更新。  
* 可选择显示指定区域和指定服务的 AWS 区域表。
* 下载 AWS 区域表 csv 文件。
* 特定区域和特定服务的搜索。
* 需求的服务集合在哪些区域全部能满足的搜索。
* 区域间服务不同的对比。  
示例页面： [Demo link](http://aws-status-check-website.s3-website.ap-northeast-2.amazonaws.com/)

## 3.如何使用本Demo

### 3.1 构建代码

```
$ git clone https://github.com/AwsDemoCenter/aws-services-list
$ cd aws-services-list
$ yarn install
$ yarn build
```
注：如有需要，请先安装yarn
### 3.2 配置服务器

选择一个部署所有服务的AWS区域，确保该区域内至少包含有上述提到的所有服务。  
以下操作使用AWS CLI，请确保已经安装
#### (1) 配置DynamoDB
使用 AWS CLI 在DynamoDB中创建一张命名为aws-services-list表，主分区键为id（String）。其余按照实际情况进行配置，一般情况下默认即可。
``` 
$  aws dynamodb create-table --table-name aws-services-list --attribute-definitions  AttributeName=id,AttributeType=S --key-schema AttributeName=id,KeyType=HASH --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5
```
#### (2) 配置 IAM Role
为lambda函数配置IAM角色,至少需要Amazon DynamoDB的读写权限以及Amazon S3的写入权限
```
#create trust relationship policy document that grants an entity permission to assume the role
$ vim policy.json
#insert following json content to policy.json
 {
 "Version": "2012-10-17",
 "Statement": {
   "Effect": "Allow",
   "Principal": {"Service": "lambda.amazonaws.com"},
   "Action": "sts:AssumeRole"
  }
}

#create a new IAM role
$ aws iam create-role --role-name service-list --assume-role-policy-document file://policy.json

#To attach a managed policy to IAM role
$ aws iam attach-role-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --role-name service-list
$ aws iam attach-role-policy --policy-arn arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess --role-name service-list
```
#### (3) 配置 Lambda
将Lambda文件夹下的commander.py和data.json文件压缩为*.zip压缩包。创建Lambda函数aws-services-list-commander，设置处理程序为commander.lambda_handler，并在基本设置中将超时时间设置为5分钟,运行语言Python 2.7。
```
$ cd lambda 
#compress file into code.zip
$ zip code.zip commander.py data.json

#create lambda function aws-services-list-commander
$ aws lambda create-function --function-name aws-services-list-commander --runtime python2.7 --role arn:aws:iam::809180791604:role/service-list --handler commander.lambda_handler --zip-file fileb://code.zip --timeout 300
```
创建Lambda函数aws-services-list-detector,超时时间设置为15秒  
创建Lambda函数aws-services-list-transmitter,超时时间设置为10秒。   
运行语言Python 2.7。
```
#create lambda function aws-services-list-detector,timeout 15s
$ zip detector.zip detector.py
$ aws lambda create-function --function-name aws-services-list-detector --runtime python2.7 --role arn:aws:iam::809180791604:role/service-list --handler detector.py --zip-file fileb://detector.zip --timeout 15

#create lambda function aws-services-list-transmitter,timeout 10s
$ zip  transmitter.zip  transmitter.py
$ aws lambda create-function --function-name aws-services-list-transmitter --runtime python2.7 --role arn:aws:iam::809180791604:role/service-list --handler  transmitter.py --zip-file fileb://transmitter.zip --timeout 10
```
#### (4) 配置 CloudWatch Events
在 CloudWatch中创建五条新的规则，分别与lambda函数aws-services-list-commander和aws-services-list-transmitter进行匹配。
其中aws-services-list-commander需要设置四条触发规则（随着服务的增多，参数会进行相应的更改），而aws-services-list-transmitter只需要设置一条无输入参数的触发规则。

aws-services-list-commander的触发规则：
```
#First Rule
#put rule 1 
$ aws events put-rule --name aws-services-list-commander-1 --schedule-expression 'cron(0 0/12 * * ? *)'

#add permission
$ aws lambda add-permission --function-name aws-services-list-commander --statement-id aws-services-list-commander-1 --action 'lambda:InvokeFunction' --principal events.amazonaws.com --source-arn YOUR_RULE_ARN

#put-targets,replace your arn
$ aws events put-targets --rule aws-services-1 \
 --targets '{"Id":"1","Arn":"YOUR_LAMBDA_ARN","Input":"{\"min_services\":0,\"max_services\":30}"}'
#Second Rule
#put rule 2
$ aws events put-rule --name aws-services-list-commander-2 --schedule-expression 'cron(4 0/12 * * ？ *)'

#add permission
$ aws lambda add-permission --function-name aws-services-list-commander --statement-id aws-services-list-commander-2 --action 'lambda:InvokeFunction' --principal events.amazonaws.com --source-arn YOUR_RULE_ARN

#put-targets,replace your arn
$aws events put-targets --rule aws-services-list-commander-2 \
 --targets '{"Id":"1","Arn":"YOUR_LAMBDA_ARN","Input":"{\"min_services\":30,\"max_services\":60}"}' 

#Third Rule
#put rule 3 
$ aws events put-rule --name aws-services-list-commander-3 --schedule-expression 'cron(8 0/12 * * ? *)'

#add permission
$ aws lambda add-permission --function-name aws-services-list-commander --statement-id aws-services-list-commander-3 --action 'lambda:InvokeFunction' --principal events.amazonaws.com --source-arn YOUR_RULE_ARN

#put-targets,replace your arn
$aws events put-targets --rule aws-services-list-commander-3 --targets "Id"="commander-3","Arn"="YOUR_LAMBDA_ARN" --targets '{"Id":"1","Arn":"YOUR_LAMBDA_ARN","Input":"{\"min_services\":60,\"max_services\":90}"}'

#Fourth Rule
#put role 4
$ aws events put-rule --name aws-services-list-commander-4 --schedule-expression 'cron(12 0/12 * * ？ *)'

#add permission
$ aws lambda add-permission --function-name aws-services-list-commander --statement-id aws-services-list-commander-4 --action 'lambda:InvokeFunction' --principal events.amazonaws.com --source-arn YOUR_RULE_ARN

#put-targets,replace your arn
$aws events put-targets --rule aws-services-list-commander-4 --targets "Id"="commander-4","Arn"="YOUR_LAMBDA_ARN" \
--targets '{"Id":"1","Arn":"YOUR_LAMBDA_ARN","Input":"{\"min_services\":90,\"max_services\":120}"}'
```

aws-services-list-transmitter的触发规则：

```
#put role:aws-services-list-transmitter
$ aws events put-rule --name aws-services-list-transmitter --schedule-expression 'cron(30 0/12 * * ？ *)'

#add permission
$ aws lambda add-permission --function-name aws-services-list-transmitter --statement-id aws-services-list-transmitter --action 'lambda:InvokeFunction' --principal events.amazonaws.com --source-arn YOUR_RULE_ARN

#put-targets,replace your arn
$ aws events put-targets --rule aws-services-list-transmitter --targets "Id"="transmitter",\
  "Arn"="YOUR_LAMBDA(aws-services-list-transmitter)_ARN" 
```
#### (5) 配置S3

创建S3存储桶，将 build 文件夹下所有文件上传到储存桶中，并设置所有文件公开化
设置S3存储桶为静态网站，主页为index.html
```
#create a new bucket:aws-services-list 
$ aws s3 mb s3://aws-services-list 

#upload directory build to s3 bucket and make it public 
$ aws s3 sync ./build s3://aws-services-list/ --grants full=uri=http://acs.amazonaws.com/groups/global/AllUsers

#confugure s3 bucket as static website
$ aws s3 website s3://aws-services-list/ --index-document index.html
```
至此，可以通过s3链接访问demo

## 4.本地运行代码
执行yarn start，即可在浏览器(chrome或IE9及以上)中进行本地访问
```
$ yarn start
```


   
[AWS Command Line Interface安装配置](https://docs.aws.amazon.com/zh_cn/cli/latest/userguide/cli-chap-welcome.html)  
[AWS Command Line Interface参考文档](https://docs.aws.amazon.com/zh_cn/cli/latest/index.html)   

