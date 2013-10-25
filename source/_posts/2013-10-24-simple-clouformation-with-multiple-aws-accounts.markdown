---
layout: post
title: "Simple Clouformation With Multiple AWS Accounts"
date: 2013-10-24 14:03
comments: true
categories: [AWS, CloudFormation, JSON]

---

In this post I'll describe how to create a simple AWS CloudFormation template
so that we can deploy stack using multiple AWS accounts.  In other words a common
JSON CloudFormation template that can be use to bring
up a stack in multiple accounts. The way we are able to do this is by
having exact copies of the [EC2 AMIs](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ComponentsAMIs.html)
on all the accounts and regions where we are deploying our stack.

With the new features from AWS including the ability to link multiple accounts,
many customers are starting to use accounts say for different departments or 
for different purposes say, production, QA, development, sales.  So, the motivation
behind this script is the need for single JSON format that works
across all accounts.

For more information on CloudFormation you can visit the [AWS Cloudformation Page](
http://aws.amazon.com/cloudformation/) as
well as the [AWS Cloudformation documentation](http://aws.amazon.com/documentation/cloudformation/) page.


First we ask for `parameters` in the CloudFormation template:

``` json cf.json
{
  "AWSTemplateFormatVersion":"2010-09-09",
  "Description":"My WebService",
  "Parameters":{
    "AwsAccount":{
      "Description":"Account: Production, or Dev",
      "Type":"String",
      "Default":"Production",
      "MinLength":"1",
      "MaxLength":"1",
      "AllowedValues":[
        "Production",
        "Dev"
      ],
      "ConstraintDescription":"Must be either 'Production', or 'Dev'"
    },
    "InstanceType":{
      "Description":"EC2 instnce type to launch",
      "Type":"String",
      "Default":"m1.large"
    },
    "MinGroupSize":{
      "Description":"Minimum number of servers to launch - Must match a multiple of avzones available in region",
      "Type":"Number",
      "Default":"3"
    },
    "MaxGroupSize":{
      "Description":"Maximum number of servers to launch - Must match a multiple of avzones available in region",
      "Type":"Number",
      "Default":"30"
    }
  },
```

Next up is the mappings definition, we have to define specific parameters for each account.
Note the accountID variable is a generic one. You have to subsitute with your specific accountIds

``` json cf.json start:33

  "Mappings":{
    "AWSAccountInfo":{
      "Production":{
        "accountId": 123456789012,
        "hostedZone":"production.mydomain.com.",
        "keypair":"production",
        "envName":"production",
        "name":"Production"
      },
      "Dev":{
        "accountId": 123456789012,
        "hostedZone":"dev.mydomain.com.",
        "keypair":"dev",
        "envName":"dev",
        "name":"Dev"
      }
    },
    "Production":{
      "us-east-1":{
        "ami":"ami-xxxxxxxx"
      },
      "us-west-1":{
        "ami":"ami-xxxxxxxx"
      },
      "us-west-2":{
        "ami":"ami-xxxxxxxx"
      }
    },
    "Dev":{
      "us-east-1":{
        "ami":"ami-xxxxxxxx"
      },
      "us-west-1":{
        "ami":"ami-xxxxxxxx"
      },
      "us-west-2":{
        "ami":"ami-xxxxxxxx"
      }
    }
  },
```

Now you want to setup your resources starting with the Autoscaling group.
Notice how on the notification configuration we specify parameters
that identify out account.

``` json cf.json start:73 mark:107-128
  "Resources":{
    "ServerGroup":{
      "Type":"AWS::AutoScaling::AutoScalingGroup",
      "Properties":{
        "AvailabilityZones":{
          "Fn::GetAZs":""
        },
        "LaunchConfigurationName":{
          "Ref":"LaunchConfig"
        },
        "MinSize":{
          "Ref":"MinGroupSize"
        },
        "MaxSize":{
          "Ref":"MaxGroupSize"
        },
        "LoadBalancerNames":[
          {
            "Ref":"ElasticLoadBalancer"
          }
        ],
        "Cooldown":"120",
        "Tags":[
          {
            "Key":"Name",
            "Value":"MyServerType",
            "PropagateAtLaunch":"true"
          },
          {
            "Key":"User",
            "Value":"Customers",
            "PropagateAtLaunch":"true"
          }
        ],
        "NotificationConfiguration":{
          "TopicARN":{
            "Fn::Join":[
              ":",
              [
                "arn:aws:sns",
                {
                  "Ref":"AWS::Region"
                },
                {
                  "Fn::FindInMap":[
                    "AWSAccountInfo",
                    {
                      "Ref":"AwsAccount"
                    },
                    "accountId"
                  ]
                },
                "notification"
              ]
            ]
          },
          "NotificationTypes":[
            "autoscaling:EC2_INSTANCE_LAUNCH",
            "autoscaling:EC2_INSTANCE_LAUNCH_ERROR",
            "autoscaling:EC2_INSTANCE_TERMINATE",
            "autoscaling:EC2_INSTANCE_TERMINATE_ERROR"
          ]
        }
      }
    },
```

Next we define our launch configuration for our instance in our autoscaling group.
Notice how we setup the environment in "UserData" to the one corresponding to the AWS account
we are using.
``` json cf.json start:138
    "LaunchConfig": {
      "Type":"AWS::AutoScaling::LaunchConfiguration",
      "Properties": {
        "KeyName": {
          "Fn::FindInMap": [
            "AWSAccountInfo",
            {
              "Ref":"AwsAccount"
            },
            "keypair"
          ]
        },
        "ImageId":{
          "Fn::FindInMap":[
            {
              "Ref":"AwsAccount"
            },
            {
              "Ref":"AWS::Region"
            },
            "ami"
          ]
        },
        "SecurityGroups":[
          {
            "Ref":"InstanceSecurityGroup"
          }
        ],
        "InstanceType":{
          "Ref":"InstanceType"
        },
        "IamInstanceProfile":{
          "Ref":"DmpInstanceProfile"
        },
        "UserData": {
          "Fn::Base64": {
             "Fn::Join": [
                 "\n",
                 [ "#!/bin/bash",
                   { "Fn::Join": [ "", [ "ENV='", { "Fn::FindInMap":[ "AWSAccountInfo", { "Ref":"AwsAccount" }, "envName" ] }, "'" ] ] }
                 ]
             ]
          }
        }
      }
    },
```

Next we define the server scale up and scale down policies for AWS Autoscale.

``` json cf.json start:184
    "ServerScaleUpPolicy":{
      "Type":"AWS::AutoScaling::ScalingPolicy",
      "Properties":{
        "AdjustmentType":"ChangeInCapacity",
        "AutoScalingGroupName":{
          "Ref":"ServerGroup"
        },
        "Cooldown":"60",
        "ScalingAdjustment": {
          "Fn::Join":[
            "", [
              {
                "Ref":"MinGroupSize"
              }
            ]
          ]
        }
      }
    },
    "ServerScaleDownPolicy":{
      "Type":"AWS::AutoScaling::ScalingPolicy",
      "Properties":{
        "AdjustmentType":"ChangeInCapacity",
        "AutoScalingGroupName":{
          "Ref":"ServerGroup"
        },
        "Cooldown":"60",
        "ScalingAdjustment": {
          "Fn::Join":[
            "", [
              "-",
              {
                "Ref":"MinGroupSize"
              }
            ]
          ]
        }
      }
    },
```

Then we define some alarms.

``` json cf.json start:223
    "CPUAlarmHigh":{
      "Type":"AWS::CloudWatch::Alarm",
      "Properties":{
        "AlarmDescription":"Scale-up if CPU > 80% for 5 minutes",
        "MetricName":"CPUUtilization",
        "Namespace":"AWS/EC2",
        "Statistic":"Average",
        "Period":"300",
        "EvaluationPeriods":"2",
        "Threshold":"70",
        "AlarmActions":[
          {
            "Ref":"ServerScaleUpPolicy"
          }
        ],
        "Dimensions":[
          {
            "Name":"AutoScalingGroupName",
            "Value":{
              "Ref":"ServerGroup"
            }
          }
        ],
        "ComparisonOperator":"GreaterThanThreshold"
      }
    },
    "CPUAlarmLow":{
      "Type":"AWS::CloudWatch::Alarm",
      "Properties":{
        "AlarmDescription":"Scale-down if CPU < 20% for 20 minutes",
        "MetricName":"CPUUtilization",
        "Namespace":"AWS/EC2",
        "Statistic":"Average",
        "Period":"300",
        "EvaluationPeriods":"4",
        "Threshold":"20",
        "AlarmActions":[
          {
            "Ref":"ServerScaleDownPolicy"
          }
        ],
        "Dimensions":[
          {
            "Name":"AutoScalingGroupName",
            "Value":{
              "Ref":"ServerGroup"
            }
          }
        ],
        "ComparisonOperator":"LessThanThreshold"
      }
    },
```

Now we define a Load Balancer.
``` json cf.json start:275
    "ElasticLoadBalancer":{
      "Type":"AWS::ElasticLoadBalancing::LoadBalancer",
      "Properties":{
        "AvailabilityZones":{
          "Fn::GetAZs":""
        },
        "Listeners":[
          {
            "LoadBalancerPort":"80",
            "InstancePort":"80",
            "Protocol":"HTTP"
          }
        ],
        "HealthCheck":{
          "Target":"HTTP:80/status",
          "HealthyThreshold":"3",
          "UnhealthyThreshold":"3",
          "Interval":"30",
          "Timeout":"5"
        }
      }
    },
```
Now Security Groups and Security Policies.
Notice the security group policy for the the
RDS backend DB.
An RDS instance can also be added to this template.

``` json cf.json start:297
    "InstanceSecurityGroup":{
      "Type":"AWS::EC2::SecurityGroup",
      "Properties":{
        "GroupDescription":"Server access",
        "SecurityGroupIngress":[
          {
            "IpProtocol":"tcp",
            "FromPort":"22",
            "ToPort":"22",
            "CidrIp":"0.0.0.0/0"
          },
          {
            "IpProtocol":"tcp",
            "FromPort":"80",
            "ToPort":"80",
            "SourceSecurityGroupOwnerId":{
              "Fn::GetAtt":[
                "ElasticLoadBalancer",
                "SourceSecurityGroup.OwnerAlias"
              ]
            },
            "SourceSecurityGroupName":{
              "Fn::GetAtt":[
                "ElasticLoadBalancer",
                "SourceSecurityGroup.GroupName"
              ]
            }
          }
        ]
      }
    },
    "RdsIngress":{
      "Type":"AWS::RDS::DBSecurityGroupIngress",
      "Properties":{
        "DBSecurityGroupName":"web-dbbackend",
        "EC2SecurityGroupName":{
          "Ref":"InstanceSecurityGroup"
        }
      }
    }

```
Finally some outputs.

``` json cf.json start:337
  },
  "Outputs":{
    "URL":{
      "Description":"The URL of the ELB",
      "Value":{
        "Fn::Join":[
          "",
          [
            "http://",
            {
              "Fn::GetAtt":[
                "ElasticLoadBalancer",
                "DNSName"
              ]
            }
          ]
        ]
      }
    }
  }
}
```

To verify that syntax of your JSON script, save the full file to something
like `cf.json` and run: `cat cf.json | python -mjson.tool`
