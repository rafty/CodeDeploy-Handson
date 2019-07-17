# CodeDeploy Blue Green Deployment

# 1. Preparation
BuildBucket nameを決める 　
e.g. yagita-for-codedeploy  

Templateを修正する  
vpc.yml -> Parameters -> S3BucketName -> Default (yagita-for-codedeploy)  

# 1. VPC & Network 構築

```bash
$ aws cloudformation create-stack \
--stack-name codedeploy-vpc \
--region ap-northeast-1 \
--template-body file://vpc.yml \
--capabilities CAPABILITY_NAMED_IAM

$ aws cloudformation update-stack \
--stack-name codedeploy-vpc \
--region ap-northeast-1 \
--template-body file://vpc.yml \
--capabilities CAPABILITY_NAMED_IAM

$ aws cloudformation delete-stack \
--stack-name codedeploy-vpc \
--region ap-northeast-1
```

# 2. Resources for blue-green

```bash
$ aws cloudformation create-stack \
--stack-name codedeploy-bg \
--region ap-northeast-1 \
--template-body file://blue-green.yml \
--capabilities CAPABILITY_NAMED_IAM

$ aws cloudformation delete-stack \
--stack-name codedeploy-bg \
--region ap-northeast-1

```

ここでWevServerにアクセスしてみる！  
SSM Sessionsでアクセスしてみる！  

1. Upload Build Image to S3 Bucket

```bash
$ cd BuildImage
$ NOW=20190718-0900
$ zip -r $NOW.zip ./*
$ aws s3 cp $NOW.zip s3://for-codedeploy-test/$NOW.zip
$ rm *.zip
```

# 説明
- GoldenAMIを持ってきてる  
  ami-0ee3055d91280485dをPublic公開してるので参照できるはず(ap-northeaset-1のみ)  
- BuildImage(Java)を持ってきてる  
- vpc.ymlでSSM Parameterを作成してる  
- CodeDeploy Agentの説明
- appspec.ymlの説明  
- userdataとの関係


# CodeDeploy の作成
- Application -> (*)DeploymentGroup  

DeploymentGroupの作成  
-> s3://for-codedeploy-test/20190718-0900.zip  

# ロールバックしてみる！ 
- 再実行  

# 講義
#### 環境変数の話
- TagとSSM 

#### ステージングの話
- Buildは1回
- 12 factor


# CodeDeployの問題点
1. CFn TemplateでBulueGreenが指定できない
1. blue-green.ymlが使えなくなる
1. GoldenImageが変えられない
1. Pipelineに加えるには厳しい。運用設計をしっかり行うこと。

問題はBlueGreenデプロイメントになってないこと


# CloudFormationで削除
-> ドリフトを体験
