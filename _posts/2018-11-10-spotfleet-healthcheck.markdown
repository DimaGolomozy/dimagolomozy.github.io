---
layout: post
title:  "ELB Health Check With AWS Spot Fleet"
date:   2018-11-10 14:00:00
description : "Tutorial on how to integrate ELB health check with Amazon Spot Fleet"
image: assets/images/spotfleet-healthcheck/spotfleet-healthcheck.png
comments: true
tags: [aws, spotfleet, health check, elb, lambda]
---
In mid-2015, AWS announced [Spot Fleet][spotfleet-annunced] to make the EC2 Spot Instance model even more useful. With the addition of a new API, Spot Fleet allowed one to launch and manage an entire fleet of Spot Instances with just one request.
  
Spot Fleet comes with some great features:
* Automatically attaching new instances to ELB/ALB
* Replacing terminated instances to maintain the Target Capacity
* Instance Weighting
 
But to better understand how Spot Fleet works, visit [How Spot Fleet Works][aws-spotfleet]
  
As great Spot Fleet is, its still has some gaps, one of them is the Health Check.  
> Spot Fleet checks the health status of the Spot Instances in the fleet every two minutes. The health status of an instance is either healthy or unhealthy. Spot Fleet determines the health status of an instance using the **status checks provided by Amazon EC2**.  

As you can see, Spot Fleet health check is based on [EC2 health check status][aws-ec2-check-status], that means no way to determine the health check by ELB.
In some cases, once the ELB marks the instance as `unhealthy`, we would like to replace it - like [AWS Auto Scaling Group][aws-asg] does. Well, that is not an option in Spot Fleet.  
But fear no more, in this blog post I will explain how it can be done with the help of a little more AWS services and some Python coding.
So lets begin...

## Architecture

We will use [AWS CloudFormation Template][aws-cloudformation-template] to deploy the following:
* An [Elastic Load Balancing][aws-elb]
* An [AWS Lambda][aws-lambda]
* An [Amazon CloudWatch][aws-cloudwatch] Alarm

For the purpose of the example, I will use a Spot Fleet behind an Application Load Balancer, that uses a health check to determine if the instance is ready for incoming traffic.
Once the instance is `unhealthy` the ALB stops sending it traffic, and I want to give it time to recover before terminating it - This is the job for... the ELB.  
When the cloudwatch alarm is triggered because of an unhealthy instances count in the ELB, a Lambda function will be executed with a python script, that will de-register the unhealthy instance and finally terminate it.
Once terminated, Spot Fleet will automatically launch a new one to maintain its Target Capacity.  

The following diagram shows how the components work together.  

![Spot Fleet With Health Check]({{ site.url }}/assets/images/spotfleet-healthcheck/spotfleet-healthcheck.png)

## Review the details
_All the code and cloudformation templates are minimal and shows only whats needed for this blog post._

#### Elastic Load Balancing
Because I want to give the instance time to recover from its `Unhealthy` state (determent by the ALB), 
I add this ELB with the same Health Check path as configured in the ALB but I give it a little more `UnhealthyThreshold`. 
This gives the instance more time before moving it to `Unhealthy` state, and finally terminates it by our Lambda.  

{% highlight json %}
{
  "SpotFleetELB": {
    "Type": "AWS::ElasticLoadBalancing::LoadBalancer",
    "Properties": {
      "LoadBalancerName": {"Fn::Sub": "${AWS::StackName}-SpotFleet-ELB"},
      "HealthCheck": {
        "Target": "HTTP:80/server/healthCheck.php",
        "HealthyThreshold": 2,
        "UnhealthyThreshold": "alb-UnhealthyThreshold + 2",
        "Interval": 20,
        "Timeout": 15
      }
    }
  }
}
{% endhighlight %}

#### Lambda functions
The Lambda function does all the magic. The details of the CloudWatch Alarm are published to the Lambda function (throughout the SNS topic),
witch uses [boto3](https://boto3.readthedocs.io) to make a couple of AWS API calls. 
The first call is to describe the all instances health of the ELB, filtering on instances that are `OutOfService`. 
The instances that pass the filtering, are then de-registered from the ALB before terminating them.  

{% highlight python %}
import boto3
import json

ALB_ARN = 'arn:aws:elasticloadbalancing:region:account-id:loadbalancer/app/SpotFleet-ALB-Name/load-balancer-id'
ELB_NAME = 'SpotFleet-ELB-Name'

def get_unhealthy_instances_ids(elb_name):
    response = boto3.client("elb").describe_instance_health(LoadBalancerName=elb_name)
    return [instance['InstanceId'] for instance in response['InstanceStates'] if instance['State'] == 'OutOfService']
		
def deregister_from_alb(instances_ids, alb_arn):
    instances_ids = [{'Id': instance_id} for instance_id in instances_ids]
	
    alb_client = boto3.client('elbv2')
    tgs = alb_client.describe_target_groups(alb_arn)['TargetGroups']

    for tg in tgs:
        alb_client.deregister_targets(
            TargetGroupArn=tg['TargetGroupArn'],
            Targets=instances_ids
        )

    waiter = alb_client.get_waiter('target_deregistered')
    for tg in tgs:
        waiter.wait(
            TargetGroupArn=tg['TargetGroupArn'],
            Targets=instances_ids
        )
        
def reset_alarm(event):
    message = json.loads(event['Records'][0]['Sns']['Message'])
    alarm = boto3.resource('cloudwatch').Alarm(message['AlarmName'])
    alarm.set_state(
        StateValue='OK',
        StateReason='Resetting alarm from Lambda'
    )

def lambda_handler(event, context):
    instances_ids = get_unhealthy_instances_ids(ELB_NAME)
    if not instances_ids:
        print("No action needed. No unhealthy instances")
        return
    
    deregister_from_alb(instances_ids, alb_arn=ALB_ARN)
    boto3.client('ec2').terminate_instances(InstanceIds=instances_ids)
    
    print("Terminated %s instances: %s", len(instances_ids), instances_ids)
    
    reset_alarm(event)
{% endhighlight %}
Pay attention to the **_reset_alarm_** function, with this function we want to reset the alarm state to `OK`.  
Because CloudWatch Alarm has no option for repeat action when the alarm is raised, we could end up with failing instances 
and the lambda won't be triggered any more. This why, by setting the alarm state back to `OK`, will cause it to trigger the action once again when the state changes to `ALARM`.

Next we will give the Lambda function permissions to be invoked from the SNS topic.
{% highlight json %}
{
  "UnhealthySNSTopicPermission": {
    "Type": "AWS::Lambda::Permission",
    "Properties": {
      "Action": "lambda:invokeFunction",
      "Principal": "sns.amazonaws.com",
      "FunctionName": {"Ref": "UnhealthyCountLambdaHandler"},
      "SourceArn": {"Ref": "SNSTopic"}
    }
  }
}
{% endhighlight %}

#### CloudWatch Alarm
Next step is to create the CloudWatch Alarm to be raised once the ELB `UnHealthyHostCount` gets above 0.
{% highlight json %}
{
  "UnHealthyAlarm": {
    "Type": "AWS::CloudWatch::Alarm",
    "Properties": {
      "ActionsEnabled": true,
      "AlarmDescription": "Alarm if instance in elb becomes unhealthy",
      "AlarmName": {"Fn::Sub": "${AWS::StackName}-elb-unhealthy-instances"},
      "ComparisonOperator": "GreaterThanThreshold",
      "Dimensions": [
        {
          "Name": "LoadBalancerName",
          "Value": {"Ref": "SpotFleetELB"}
        }
      ],
      "Period": "60",
      "EvaluationPeriods": 2,
      "MetricName": "UnHealthyHostCount",
      "Namespace": "AWS/ELB",
      "Statistic": "Maximum",
      "Threshold": "0",
      "AlarmActions": [{"Ref": "SNSTopic"}]
    }
  }
}
{% endhighlight %}

#### SNS Topic
And now, the SNS topic that connects the CloudWatch Alarm and the Lambda function.  
{% highlight json %}
{
  "SNSTopic": {
    "Type": "AWS::SNS::Topic",
    "Properties": {
      "TopicName": {"Fn::Sub": "${AWS::StackName}-Unhealthy-Instances"},
      "DisplayName": "Unhealthy-Instances",
      "Subscription": [
        {
          "Protocol": "lambda",
          "Endpoint": {"Fn::GetAtt": ["UnhealthyCountLambdaHandler","Arn"]}
        }
      ]
    }
  }
}
{% endhighlight %}

#### Spot Fleet request
Finally, the Spot Fleet request, that uses an Application Load Balancer to send traffic to the connected instances. 
And uses the Elastic Load Balancing for the Health Check (I could use the ALB for that, but again, I wanted to give the instances time to recover)
{% highlight json %}
{
  "V2VSpotFleet": {
    "Type": "AWS::EC2::SpotFleet",
    "Properties": {
      "SpotFleetRequestConfigData": {
        "Type": "maintain",
        "TargetCapacity": 10,
        "AllocationStrategy": "lowestPrice",
        "IamFleetRole": {"Ref": "IamFleetRole"},
        "LaunchTemplateConfigs": [
          {
            "LaunchTemplateSpecification": {
              "LaunchTemplateId": {"Ref": "SpotFleetLaunchTemplate"},
              "Version": "1"
            },
            "Overrides": []
          }
        ],
        "LoadBalancersConfig": {
          "TargetGroupsConfig": {
            "TargetGroups": [
              {"Arn": {"Ref": "AlbTargetGroup"}}
            ]
          },
          "ClassicLoadBalancersConfig": {
            "ClassicLoadBalancers": [
              {"Name": {"Ref": "SpotFleetELB"}}
            ]
          }
        }
      }
    }
  }
}
{% endhighlight %}

## Test
After launching the CloudFormation stack. I want to simulate an `unhealthy` instance. For that I logged into a random instance from the Spot Fleet and stopped my application (the application that supposed to answer to the Health Check path).
That will cause the ALB to mark the instance as `unhealthy` and traffic wont be sent to it. Then the ELB will mark the instance as `OutOfService`, and that will trigger the CloudWatch Alarm, that will invoke the Lambda function, that will de-register it from the ALB and finally will terminate the instance.


[spotfleet-annunced]: https://aws.amazon.com/blogs/aws/amazon-ec2-spot-fleet-api-manage-thousands-of-instances-with-one-request
[aws-spotfleet]: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-fleet.html
[aws-ec2-check-status]: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitoring-system-instance-status-check.html
[aws-asg]: https://docs.aws.amazon.com/autoscaling/ec2/userguide/GettingStartedTutorial.html
[aws-cloudformation-template]: https://aws.amazon.com/cloudformation/aws-cloudformation-templates
[aws-elb]: https://aws.amazon.com/elasticloadbalancing
[aws-lambda]: https://aws.amazon.com/lambda
[aws-cloudwatch]: https://aws.amazon.com/cloudwatch 
