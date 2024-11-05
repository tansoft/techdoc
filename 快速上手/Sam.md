
# 安装

Mac:

```bash
brew tap aws/tap
brew install aws-sam-cli
```

General:

```bash
wget https://github.com/aws/aws-sam-cli/releases/latest/download/aws-sam-cli-linux-x86_64.zip
unzip aws-sam-cli-linux-x86_64.zip -d sam-installation
sudo ./sam-installation/install --update
sam --version
```

# Quick Start

* https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-specification-template-anatomy.html

* Type: 'AWS::Serverless::Function' 表示创建Lambda函数类型的资源。
* 还支持：AWS::Serverless::Api对应API Gateway, AWS::Serverless::SimpleTable对应DynamoDB
* Handler: app.lambda_handler 表示Lambda的handler函数，格式为文件名 + 入口函数名。这里我们的文件名为app.py，入口函数为def lambda_hander()。
* Runtime表示Lambda的执行语言环境
* CodeUri：代码目录。我们的python代码放在src/目录下
* MemorySize, Timeout：lambda的内存及超时时间配置

```bash
mkdir sam-app && cd sam-app
mkdir src
touch src/app.py
touch template.yaml

# src/app.py
import json

print("Loading function")

def lambda_handler(event, context):
    return "Hello world"

# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: 'AWS::Serverless-2016-10-31'
Description: A starter AWS Lambda function.
Resources:
  helloworldpython3:
    Type: 'AWS::Serverless::Function'
    Properties:
      Handler: app.lambda_handler
      Runtime: python3.7
      CodeUri: src/
      Description: A starter AWS Lambda function.
      MemorySize: 128
      Timeout: 3

```

* sam build 生成运行环境，每次代码修改后也需要重新运行
* sam local invoke 本地环境测试，会下载运行环境的docker镜像，可以通过 docker images 查看
* sam package 打包成 CloudFormation 模版
* sam depoly --guided 部署

* 也可以用 sam init 生成环境，包括测试事件也会生成

```bash
sam local invoke -e events/event.json 测试
echo '{"message": "Hey, are you there?" }' | sam local invoke --event - "HelloWorldFunction"
```

* sam local start-api 本地启动 api gateway 测试，通常是 3000 端口
* sam docs 打开线上文档链接
* sam list
* sam delete
* sam publish 发布公开模版

## 创建ddb

* 通过环境变量 TABLE_NAME 引用ddb
* SAM具有创建DynamoDB表的能力，但只能创建primary key类型的表，如果要使用primary + sort key类型的表，需要提前进行创建

```yaml
# SAM FILE
AWSTemplateFormatVersion: '2010-09-09'
Transform: 'AWS::Serverless-2016-10-31'
Description: A starter AWS Lambda function.
Resources:
  helloworldpython3:
    Type: 'AWS::Serverless::Function'
    Properties:
      Handler: app.lambda_handler
      Runtime: python3.7   # 版本和当前环境Python保持一致
      CodeUri: hello_world/   # 代码目录在hello_world下面
      Description: A starter AWS Lambda function.
      MemorySize: 128
      Timeout: 3
      Environment:  # 传入环境变量
        Variables:
          TABLE_NAME: !Ref Table   
          REGION_NAME: !Ref AWS::Region
      Events:  # API Gateway设置
        HelloWorldSAMAPI:
          Type: Api
          Properties:
            Path: /hello
            Method: GET
      Policies:  # Lambda需要有访问DynamoDB CRUD权限
        - DynamoDBCrudPolicy:
            TableName: !Ref Table

  Table:  # 创建DynamoDB表
    Type: AWS::Serverless::SimpleTable
    Properties:
      PrimaryKey: # Primary设置
        Name: greeting
        Type: String
      ProvisionedThroughput:  # DynamoDB RCU/WCU配置
        ReadCapacityUnits: 2
        WriteCapacityUnits: 2
```

# AWS Application Composer

一个图形化的设计器，使用多个AWS服务创建无服务器应用。开发者新创建无服务器应用时，会面临一个学习曲线，需要学习很多AWS服务。他们需要学习这些服务的配置，然后写IaC来创建应用

Application Composer会默认设置好lambda的一些属性，比如开启tracing(X-Ray)和设置超时时间。可以在CloudFormation中更改这些属性或在resource properties 页面更改

如果两个服务需要进行交互，Application Composer会为Lambda自动设置好IAM policy、环境变量、event subscriptions 等属性。

# Workflow

使用Step Function Workflow Studio创建State Machine, 用UI工具来设计Workflow

* 进入Step Functions服务，点击Create state machine， 并进入下一步
* 先将Pass类型的Flow拖入框框中，加一点东西
* 进行测试，点击Start execution， 在弹出的页面中继续点击Start Execution

# Cognito

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  cognito-webapp
  Amazon Cognito Workshop
  
Resources:
  CognitoWebApp:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: !Sub ${AWS::StackName}-CognitoWebApp
      CodeUri: web-app/
      Handler: run.sh
      Runtime: nodejs18.x
      MemorySize: 1024
      Timeout: 3
      Architectures:
        - x86_64
      Environment:
        Variables:
          AWS_LAMBDA_EXEC_WRAPPER: /opt/bootstrap
          RUST_LOG: info
      Layers:
        - !Sub arn:aws:lambda:${AWS::Region}:753240598075:layer:LambdaAdapterLayerX86:17
      Events:
        RootPath:
          Type: Api
          Properties:
            Path: /
            Method: ANY
        AnyPath:
          Type: Api
          Properties:
            Path: /{proxy+}
            Method: ANY

Outputs:
  CognitoWebAppURL:
    Description: "Cognito Workshop Web App URL"
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.${AWS::URLSuffix}/Prod/"
```

# CloudFormation

```yaml
Resources: 
  LambdaExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
        - Effect: Allow
          Principal:
            Service:
            - lambda.amazonaws.com
          Action:
          - sts:AssumeRole
      Path: "/"
      Policies:
      - PolicyName: root
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
          - Effect: Allow
            Action:
            - "s3:*"
            Resource: "*"
          - Effect: Allow
            Action:
            - "logs:CreateLogGroup"
            - "logs:CreateLogStream"
            - "logs:PutLogEvents"
            Resource: "*"
  ListBucketsS3Lambda: 
    Type: "AWS::Lambda::Function"
    Properties: 
      Handler: "index.handler"
      Role: 
        Fn::GetAtt: 
          - "LambdaExecutionRole"
          - "Arn"
      Runtime: "python3.7"
      Code: 
        ZipFile: |
          import boto3
          # Create an S3 client
          s3 = boto3.client('s3')
          def handler(event, context):
            # Call S3 to list current buckets
            response = s3.list_buckets()
            # Get a list of all bucket names from the response
            buckets = [bucket['Name'] for bucket in response['Buckets']]
            # Print out the bucket list
            print("Bucket List: %s" % buckets)
            return buckets
```

Lambda 在 S3 上，可以通过 S3ObjectVersionParam 参数指定版本，CloudFormation才可以检测到更新

```yaml
Parameters:
  S3BucketParam:
    Type: String
  S3KeyParam:
    Type: String
  S3ObjectVersionParam:
    Type: String
Resources: 
  LambdaExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
        - Effect: Allow
          Principal:
            Service:
            - lambda.amazonaws.com
          Action:
          - sts:AssumeRole
      Path: "/"
      Policies:
      - PolicyName: root
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
          - Effect: Allow
            Action:
            - "s3:*"
            Resource: "*"
          - Effect: Allow
            Action:
            - "logs:CreateLogGroup"
            - "logs:CreateLogStream"
            - "logs:PutLogEvents"
            Resource: "*"
  ListBucketsS3Lambda: 
    Type: "AWS::Lambda::Function"
    Properties: 
      Handler: "index.handler"
      Role: 
        Fn::GetAtt: 
          - "LambdaExecutionRole"
          - "Arn"
      Runtime: "python3.7"
      Code: 
        S3Bucket: 
          Ref: S3BucketParam
        S3Key: 
          Ref: S3KeyParam
        S3ObjectVersion:
          Ref: S3ObjectVersionParam
```
