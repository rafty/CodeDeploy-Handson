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

13:20

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

13:26

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

7/19
13:29

# 10. CodeDeployアプリケーションの作成
name: ec

# 11. CodeDeployデプロイグループの作成
デプロイグループ名: Webservers

7/19
13:30

# 12. CodeDeployデプロイグループの実行
[デプロイの作成]ボタン
-> s3://for-codedeploy-test/20190718-0900.zip  
-> s3://yagita-for-codedeploy-ap-northeast-1/20190718-0900.zip


7/19
13:39

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


---

# 動作解析

CreateAutoScalingGroup
-> CodeDeploy_Webservers_d-HQU3QSJWA
-> "launchConfigurationName": "ec-websvr-dev-lcfg",

      "requestParameters": {
        "availabilityZones": [
          "ap-northeast-1a",
          "ap-northeast-1c"
        ],
        "healthCheckGracePeriod": 0,
        "maxSize": 2,
        "minSize": 0,
        "desiredCapacity": 0,
        "launchConfigurationName": "ec-websvr-dev-lcfg",
        "terminationPolicies": [
          "Default"
        ],
        "tags": [
          {
            "propagateAtLaunch": true,
            "key": "Environment",
            "value": "dev"
          },
          {
            "propagateAtLaunch": true,
            "key": "ProjectName",
            "value": "ec"
          },
          {
            "propagateAtLaunch": true,
            "key": "RoleName",
            "value": "websvr"
          },
          {
            "key": "CodeDeployProvisioningDeploymentId",
            "value": "d-HQU3QSJWA"
          }
        ],
        "defaultCooldown": 300,
        "newInstancesProtectedFromScaleIn": false,
        "vPCZoneIdentifier": "subnet-0b6367f01f2f57f88,subnet-03866f7c02d0656a3",
        "autoScalingGroupName": "CodeDeploy_Webservers_d-HQU3QSJWA",
        "healthCheckType": "EC2"
      },

PutScalingPolicy       -> CodeDeploy_Webservers_d-HQU3QSJWA

      "requestParameters": {
        "scalingAdjustment": -1,
        "adjustmentType": "ChangeInCapacity",
        "autoScalingGroupName": "CodeDeploy_Webservers_d-HQU3QSJWA",
        "policyType": "SimpleScaling",
        "policyName": "codedeploy-bg-WebServerScaleDownPolicy-1VF4TFWSN0J56",
        "cooldown": 60
      },

PutNotificationConfiguration -> 

      "requestParameters": {
        "notificationTypes": [
          "autoscaling:EC2_INSTANCE_LAUNCH_ERROR",
          "autoscaling:EC2_INSTANCE_TERMINATE_ERROR",
          "autoscaling:EC2_INSTANCE_TERMINATE",
          "autoscaling:EC2_INSTANCE_LAUNCH",
          "autoscaling:TEST_NOTIFICATION"
        ],
        "topicARN": "arn:aws:sns:ap-northeast-1:338456725408:ec-websvr-dev-asg-topic",
        "autoScalingGroupName": "CodeDeploy_Webservers_d-HQU3QSJWA"
      },

UpdateAutoScalingGroup -> 

      "requestParameters": {
        "minSize": 2,
        "desiredCapacity": 2,
        "autoScalingGroupName": "CodeDeploy_Webservers_d-HQU3QSJWA"
      },

SuspendProcesses
 Auto Scaling プロセスの実行を防止する
 
      "requestParameters": {
        "autoScalingGroupName": "CodeDeploy_Webservers_d-HQU3QSJWA",
        "scalingProcesses": [
          "AddToLoadBalancer",
          "AlarmNotification",
          "ScheduledActions"
        ]
      },


新インスタンス
i-0cdecf6e1c3e337ac
i-0335d33e72cde039d

RegisterTargets

      "requestParameters": {
        "targets": [
          {
            "id": "i-0335d33e72cde039d"
          }
        ],
        "targetGroupArn": "arn:aws:elasticloadbalancing:ap-northeast-1:338456725408:targetgroup/ec-websvr-dev-tg/3a907d569cdf4b09"
      },

RegisterTargets

      "requestParameters": {
        "targets": [
          {
            "id": "i-0cdecf6e1c3e337ac"
          }
        ],
        "targetGroupArn": "arn:aws:elasticloadbalancing:ap-northeast-1:338456725408:targetgroup/ec-websvr-dev-tg/3a907d569cdf4b09"
      },

DeregisterTargets

      "requestParameters": {
        "targetGroupArn": "arn:aws:elasticloadbalancing:ap-northeast-1:338456725408:targetgroup/ec-websvr-dev-tg/3a907d569cdf4b09",
        "targets": [
          {
            "id": "i-01b517103f6099228"
          }
        ]
      },

DeregisterTargets

      "requestParameters": {
        "targetGroupArn": "arn:aws:elasticloadbalancing:ap-northeast-1:338456725408:targetgroup/ec-websvr-dev-tg/3a907d569cdf4b09",
        "targets": [
          {
            "id": "i-0a4bd2696c4f4fc66"
          }
        ]
      },

ResumeProcesses

      "requestParameters": {
        "autoScalingGroupName": "CodeDeploy_Webservers_d-HQU3QSJWA",
        "scalingProcesses": [
          "ScheduledActions",
          "AddToLoadBalancer",
          "AlarmNotification"
        ]
      },

DeleteLifecycleHook

      "requestParameters": {
        "lifecycleHookName": "CodeDeploy-managed-automatic-launch-deployment-hook-Webservers-dbaae138-28dd-47f7-a8fa-2d2f9bf56dd3",
        "autoScalingGroupName": "ec-websvr-dev-asg"
      },

PutLifecycleHook

      "requestParameters": {
        "notificationMetadata": "19f7ca08-3817-4e31-bf35-8214d94ca9f1",
        "lifecycleTransition": "autoscaling:EC2_INSTANCE_LAUNCHING",
        "autoScalingGroupName": "CodeDeploy_Webservers_d-HQU3QSJWA",
        "notificationTargetARN": "arn:aws:sqs:ap-northeast-1:528078681758:razorbill-ap-northeast-1-prod-default-autoscaling-lifecycle-hook",
        "defaultResult": "ABANDON",
        "lifecycleHookName": "CodeDeploy-managed-automatic-launch-deployment-hook-Webservers-8817ca40-e6bd-4d78-b31c-56313137ee25",
        "heartbeatTimeout": 600
      },

DeleteAutoScalingGroup

      "requestParameters": {
        "forceDelete": true,
        "autoScalingGroupName": "ec-websvr-dev-asg"
      },




# AWS CLI

aws autoscaling describe-auto-scaling-groups \
--auto-scaling-group-names CodeDeploy_Webservers_d-HQU3QSJWA

```json
{
    "AutoScalingGroups": [
        {
            "AutoScalingGroupARN": "arn:aws:autoscaling:ap-northeast-1:338456725408:autoScalingGroup:df9a8320-1030-420e-bdc7-d0cb968e168e:autoScalingGroupName/CodeDeploy_Webservers_d-HQU3QSJWA", 
            "TargetGroupARNs": [], 
            "SuspendedProcesses": [], 
            "DesiredCapacity": 2, 
            "Tags": [
                {
                    "ResourceType": "auto-scaling-group", 
                    "ResourceId": "CodeDeploy_Webservers_d-HQU3QSJWA", 
                    "PropagateAtLaunch": true, 
                    "Value": "d-HQU3QSJWA", 
                    "Key": "CodeDeployProvisioningDeploymentId"
                }, 
                {
                    "ResourceType": "auto-scaling-group", 
                    "ResourceId": "CodeDeploy_Webservers_d-HQU3QSJWA", 
                    "PropagateAtLaunch": true, 
                    "Value": "dev", 
                    "Key": "Environment"
                }, 
                {
                    "ResourceType": "auto-scaling-group", 
                    "ResourceId": "CodeDeploy_Webservers_d-HQU3QSJWA", 
                    "PropagateAtLaunch": true, 
                    "Value": "ec", 
                    "Key": "ProjectName"
                }, 
                {
                    "ResourceType": "auto-scaling-group", 
                    "ResourceId": "CodeDeploy_Webservers_d-HQU3QSJWA", 
                    "PropagateAtLaunch": true, 
                    "Value": "websvr", 
                    "Key": "RoleName"
                }
            ], 
            "EnabledMetrics": [], 
            "LoadBalancerNames": [], 
            "AutoScalingGroupName": "CodeDeploy_Webservers_d-HQU3QSJWA", 
            "DefaultCooldown": 300, 
            "MinSize": 2, 
            "Instances": [
                {
                    "ProtectedFromScaleIn": false, 
                    "AvailabilityZone": "ap-northeast-1c", 
                    "InstanceId": "i-0335d33e72cde039d", 
                    "HealthStatus": "Healthy", 
                    "LifecycleState": "InService", 
                    "LaunchConfigurationName": "ec-websvr-dev-lcfg"
                }, 
                {
                    "ProtectedFromScaleIn": false, 
                    "AvailabilityZone": "ap-northeast-1a", 
                    "InstanceId": "i-0cdecf6e1c3e337ac", 
                    "HealthStatus": "Healthy", 
                    "LifecycleState": "InService", 
                    "LaunchConfigurationName": "ec-websvr-dev-lcfg"
                }
            ], 
            "MaxSize": 2, 
            "VPCZoneIdentifier": "subnet-0b6367f01f2f57f88,subnet-03866f7c02d0656a3", 
            "HealthCheckGracePeriod": 0, 
            "TerminationPolicies": [
                "Default"
            ], 
            "LaunchConfigurationName": "ec-websvr-dev-lcfg", 
            "CreatedTime": "2019-07-19T04:30:30.581Z", 
            "AvailabilityZones": [
                "ap-northeast-1a", 
                "ap-northeast-1c"
            ], 
            "HealthCheckType": "EC2", 
            "NewInstancesProtectedFromScaleIn": false
        }
    ]
}
```