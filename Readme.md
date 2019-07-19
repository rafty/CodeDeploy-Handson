# CodeDeploy Blue Green Deployment

# 1. Preparation
BuildImageBucketの名前nameを決める  
e.g. yagita-for-codedeploy-ap-northeast-1

yagitaの部分を自分の名前に変更する
```bash
$ MYNAME=yagita
$ aws cloudformation create-stack \
--stack-name codedeploy-preparation \
--region ap-northeast-1 \
--template-body file://preparation.yml \
--capabilities CAPABILITY_NAMED_IAM \
--parameters \
ParameterKey=S3BucketPrefix,ParameterValue=$MYNAME
```

output: yagita-for-codedeploy-ap-northeast-1  


# 2. VPC & Network 構築

```bash
$ MYMAIL=yagita.takashi@gmail.com

$ aws cloudformation create-stack \
--stack-name codedeploy-vpc \
--region ap-northeast-1 \
--template-body file://vpc.yml \
--capabilities CAPABILITY_NAMED_IAM \
--parameters \
ParameterKey=OperatorEMail,ParameterValue=$MYMAIL


$ aws cloudformation update-stack \
--stack-name codedeploy-vpc \
--region ap-northeast-1 \
--template-body file://vpc.yml \
--capabilities CAPABILITY_NAMED_IAM \
--parameters \
ParameterKey=OperatorEMail,ParameterValue=$MYMAIL


$ aws cloudformation delete-stack \
--stack-name codedeploy-vpc \
--region ap-northeast-1
```

# 3. Resources for blue-green

```bash
$ PROJECTNAME=ec
$ ROLENAME=websvr
$ ENV=dev

$ aws cloudformation create-stack \
--stack-name codedeploy-bg \
--region ap-northeast-1 \
--template-body file://blue-green.yml \
--capabilities CAPABILITY_NAMED_IAM \
--parameters \
ParameterKey=ProjectName,ParameterValue=$PROJECTNAME \
ParameterKey=RoleName,ParameterValue=$ROLENAME \
ParameterKey=Environment,ParameterValue=$ENV

$ aws cloudformation delete-stack \
--stack-name codedeploy-bg \
--region ap-northeast-1

```

# 4. Upload Build Image to S3 Bucket

```bash
$ cd BuildImage
$ NOW=20190718-0900
$ zip -r $NOW.zip ./*
$ aws s3 cp $NOW.zip s3://$MYNAME-for-codedeploy-ap-northeast-1/$NOW.zip
$ rm *.zip
```

# 4. 動作確認

ここでWevServerにアクセスしてみる！ -> 502 Bad Gateway
SSM Sessionsでアクセスしてみる！  


# 説明
- GoldenAMIを持ってきてる  
  ami-0ee3055d91280485dをPublic公開してるので参照できるはず(ap-northeaset-1のみ)  
- BuildImage(Java) ->  BuildImage/target/SampleMavenTomcatApp.war 
- vpc.ymlでSSM Parameterを作成してる  
- CodeDeploy Agentとappspec.ymlの説明  
- userdataとの関係

# 10. CodeDeployアプリケーションの作成
name: ec

# 11. CodeDeployデプロイグループの作成
デプロイグループ名: Webservers

# 12. CodeDeployデプロイグループの実行
[デプロイの作成]ボタン
-> s3://for-codedeploy-test/20190718-0900.zip  
-> s3://yagita-for-codedeploy-ap-northeast-1/20190718-0900.zip


# 13. ロールバックしてみる！ 
- 再実行  

# 説明
#### 環境変数の話
- Tag&SSM&userdata 


# CodeDeployの問題点
1. CFn TemplateでBulueGreenが指定できない
1. blue-green.ymlが使えなくなる
1. GoldenImageが変えられない
1. Pipelineに加えるには厳しい。運用設計をしっかり行うこと。

# 14. CloudFormationで削除
-> ドリフトを体験
