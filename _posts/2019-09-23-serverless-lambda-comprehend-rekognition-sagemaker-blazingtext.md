---
layout: post
title:  "商品类别识别及图片审核方案"
toc: true
---



## 概述

电商网站通常有大量的商品信息，通常包括两类信息文本（商品名称，描述）和图片信息。根据某些国家及地区的法律及法规要求，成人用品在某些区域销售属于禁售品类，第三发卖家有时通过删除关键字的方法绕过网站基于关键字匹配禁售机制，导致法律风险。本文描述了基于AWS Comprehend、SageMaker自带的内建BlazingText算法及Facebook Fasttext算法和AWS ECS Fargate自建方案来实现通过对商品文本的基于NLP（自然语言处理）的品类自动判别，从而不依赖个别关键字。同时使用AWS的Rekognition服务来轻松实现对含NSFW（Not Safe for Work）内容的商品图片的审核。



## Demo网站

- 网站主页 [http://mb3.zjutdp.club.s3-website-us-east-1.amazonaws.com/](http://mb3.zjutdp.club.s3-website-us-east-1.amazonaws.com/)
- 模拟卖家创建商品页面 [http://mb3.zjutdp.club.s3-website-us-east-1.amazonaws.com/createproduct.html](http://mb3.zjutdp.club.s3-website-us-east-1.amazonaws.com/createproduct.html)
- 模拟电商商品审核页面 [http://mb3.zjutdp.club.s3-website-us-east-1.amazonaws.com/moderation.html](http://mb3.zjutdp.club.s3-website-us-east-1.amazonaws.com/moderation.html)

> 因为Demo的数据存储在DynamoDB的表上应用了2个小时TTL设置，所以数据两个小时候会自动清空

## 架构总览

![总体架构](https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/aws-NLP/ArchAll.png "总体架构")

从上图可以看出，这个方案分别从AWS AI服务的三个层面：应用服务层（Amazon Comprehend)、平台服务层（Amazon SageMaker自带BlazingText算法）及自定义算法和容器来实现根据商品文本的自然语言分类模型来识别是否是成人商品，同时使用应用服务层的Rekognition Image服务来实现对商品图片是否含有NSFW的内容审核。

架构整体上是云原生、无服务器且基于事件驱动，各个功能组件实现松耦合。涵盖了从最简单的AWS AI服务的API调用到使用SageMaker内置算法，最后从算法到容器定义托管的全自定义方案，从而达到从AWS AI服务不同层面来实现商品类别识别的需求。

### 图片审核架构

![图片审核架构](https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/aws-NLP/ArchRek.png "图片审核架构")





### Comprehend自定义分类器相关部分架构

![图片审核架构](https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/aws-NLP/ArchCCC.png "图片审核架构")



### SageMaker BlazingText算法相关部分架构

![图片审核架构](https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/aws-NLP/ArchSageMaker.png "图片审核架构")



### 利用Facebook Fasttext算法及自构建ECS Fargate推理服务相关部分架构

![图片审核架构](https://aws-demo-center.s3-ap-southeast-1.amazonaws.com/demopic/aws-NLP/ArchECS.png "图片审核架构")



## 部分代码展示

#### 图片审核Lambda部分

```python
import json
import urllib.parse
import boto3

print('Loading function')

dynamodb = boto3.client('dynamodb')

def lambda_handler(event, context):
    print("Received event: " + json.dumps(event, indent=2))

    # Get the object from the event and show its content type
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = urllib.parse.unquote_plus(event['Records'][0]['s3']['object']['key'], encoding='utf-8')
    
    s3 = boto3.resource('s3')
    object_acl = s3.ObjectAcl(bucket, key)
    
    # Update image file to public readable so that website could access it
    response = object_acl.put(ACL='public-read')

    # Invoke Rekognition image moderation API
    rek = boto3.client('rekognition')
    rek_resp = rek.detect_moderation_labels(
                    Image={
                        'S3Object': {
                            'Bucket': bucket,
                            'Name': key,
                        }
                    },
                    MinConfidence=20
                )

    # Flush Rek result into DDB
    product_id = key[key.find('/')+1:]
    response = dynamodb.update_item(
        TableName='Products',
        Key={
            'id': {
                'S':  product_id
            }
        },
        UpdateExpression="set nsfw=:nsfw",
        
        # Update only if record exists by using condition expression
        ConditionExpression='id = :id',
        ExpressionAttributeValues={
            ':id' : {
                'S': product_id
            },
            ':nsfw': {
                'S': json.dumps(rek_resp['ModerationLabels'])
            }
        },
        ReturnValues="UPDATED_NEW"
    )
    
    print("DDB Response: " + json.dumps(response))
    
    return {
        'statusCode': 200,
        'headers' : {
            "Access-Control-Allow-Origin": "*"
        },
        'body': {
            "rek" : json.dumps(rek_resp),
            'ddb' : json.dumps(response)
        }
    }
```

#### 提交Comprehend自定义分类器分析任务Lambda代码

```python
import os
import os.path
import sys

# envLambdaTaskRoot = os.environ["LAMBDA_TASK_ROOT"]
# print("LAMBDA_TASK_ROOT env var:"+os.environ["LAMBDA_TASK_ROOT"])
# print("sys.path:"+str(sys.path))

# sys.path.insert(0,envLambdaTaskRoot+"/NewBotoVersion")
# print("sys.path:"+str(sys.path))
import botocore
import boto3
import json

print(boto3.__version__)

CCC_BUCKET_NAME = os.environ['CCC_BUCKET_NAME']
CCC_BUCKET_PREFIX = os.environ['CCC_BUCKET_PREFIX']
CCC_BUCKET_PREFIX_OUTPUT = os.environ['CCC_BUCKET_PREFIX_OUTPUT']

s3 = boto3.client('s3')
comprehend = boto3.client('comprehend')

def lambda_handler(event, context):
    #print("Received event: " + json.dumps(event, indent=2))
    for record in event['Records']:
        # print(record['eventID'])
        # print(record['eventName'])
        
        # Submit a Comprehend custom classification analysis job to S3 bucket when a new product created
        if record['eventName'] == 'INSERT':
            print(record['dynamodb']['NewImage']['id']['S'])
            print(record['dynamodb']['NewImage']['name']['S'])
            #print(record['dynamodb']['NewImage']['desc']['S'])
            
            id = record['dynamodb']['NewImage']['id']['S']
            text = record['dynamodb']['NewImage']['name']['S'] + record['dynamodb']['NewImage']['desc']['S']
            file = CCC_BUCKET_PREFIX + '/' + id
            
            #response = s3.get_object(Bucket=bucket, Key=key)
            s3_resp = s3.put_object(Bucket=CCC_BUCKET_NAME, Key=file, Body=text)
            
            start_response = comprehend.start_document_classification_job(
                JobName=id,
                InputDataConfig={
                    'S3Uri': 's3://{}/{}'.format(CCC_BUCKET_NAME, file),
                    'InputFormat': 'ONE_DOC_PER_FILE'
                },
                OutputDataConfig={
                    'S3Uri': 's3://{}/{}'.format(CCC_BUCKET_NAME, CCC_BUCKET_PREFIX_OUTPUT)
                },
                #DataAccessRoleArn='arn:aws:iam::124456859051:role/service-role/GetProductDetail-role-eselzlwx',
                #DataAccessRoleArn='arn:aws:iam::124456859051:role/Admin',
                DataAccessRoleArn='arn:aws:iam::124456859051:role/service-role/AmazonComprehendServiceRole-AmazonComprehendServiceRole2-',
                DocumentClassifierArn='arn:aws:comprehend:us-east-1:124456859051:document-classifier/AdultProduct'
            )
            
            print("Start response: %s\n", start_response)
            
            # Check the status of the job
            #describe_response = comprehend.describe_document_classification_job(JobId=start_response['JobId'])
            #print("Describe response: %s\n", describe_response)

    return 'Successfully processed {} records.'.format(len(event['Records']))

```



#### 在SageMaker的Notebook中使用打好标签的数据来训练、部署分类识别服务

```python
import json
import boto3
import pandas
import sagemaker
from sagemaker import get_execution_role

sess = sagemaker.Session()
role = get_execution_role()

# 设置数据集
bucket = 'zjutdp-mb3-data' 
s3_test_data = 's3://{}/{}'.format(bucket, 'labeled/flipkart-blazingtext/flipkart-test.csv')
s3_train_data = 's3://{}/{}'.format(bucket, 'labeled/flipkart-blazingtext/flipkart-train.csv')
s3_validation_data = 's3://{}/{}'.format(bucket, 'labeled/flipkart-blazingtext/flipkart-valid.csv')

s3_output_location = 's3://{}/{}/output'.format(bucket, 'labeled/flipkart-blazingtext')

region_name = boto3.Session().region_name

# 设置算法容器
container = sagemaker.amazon.amazon_estimator.get_image_uri(region_name, "blazingtext", "latest")
print('Using SageMaker BlazingText container: {} ({})'.format(container, region_name))

# 建立SageMaker模型
bt_model = sagemaker.estimator.Estimator(container,
                                         role, 
                                         train_instance_count=1, 
                                         train_instance_type='ml.c4.xlarge',
                                         train_volume_size = 30,
                                         train_max_run = 360000,
                                         input_mode= 'File',
                                         output_path=s3_output_location,
                                         sagemaker_session=sess)

# 设置超参
bt_model.set_hyperparameters(mode="supervised",
                            epochs=20,
                            min_count=20,
                            learning_rate=0.1,
                            vector_dim=150,
                            early_stopping=True,
                            patience=4,
                            min_epochs=5,
                            word_ngrams=2)

# 设置训练验证数据集
train_data = sagemaker.session.s3_input(s3_train_data, distribution='FullyReplicated', 
                        content_type='text/plain', s3_data_type='S3Prefix')
validation_data = sagemaker.session.s3_input(s3_validation_data, distribution='FullyReplicated', 
                             content_type='text/plain', s3_data_type='S3Prefix')
data_channels = {'train': train_data, 'validation': validation_data}

# 训练模型
bt_model.fit(inputs=data_channels, logs=True)

# 部署模型
text_classifier = bt_model.deploy(initial_instance_count = 1,instance_type = 'ml.m4.xlarge')

# 读取测试数据
df = pandas.read_csv(s3_test_data, header = None)

# 取出前X（100）个产品文本进行测试
n = 100
sentences = df[1][:n].values.tolist()
payload = {"instances" : sentences}

# 使用部署好的文本分类服务进行推理
response = text_classifier.predict(json.dumps(payload))
predictions = json.loads(response)

# 验证前x的产品文本分类结果和测试数据集合中的相同
assert df[0][:n].values.tolist() == [p['label'][0] for p in predictions]


```

