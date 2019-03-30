---
layout: post
title:  "How I stopped using Spotinst Elastigroup"
description: "Use AWS AutoScalingGroup with Spots and OnDemand for fallback and not Spotinst Elastigroup"
image: assets/images/folder/image.png
comments: true
date: 2019-01-10 14:00:00
tags: [AWS, Spotinst, Elastigroup, ASG, Spots, OnDemand, OD, Fallback, CloudFormation]
---

[Spotinst Elastigroup][spotinst-elastigroup] brought lots of nice features to the table. They have the ability to define a "single" group with multiple instance types, Spots and OnDemand, and can predict Spot behavior, capacity trends, pricing, and balance the capacity accordingly.

BUT, what if we don't need all those fancy features, and we just want ASG with Spots and OnDemand instances for fallback (and to lower the "cloud" costs) - well this post is just the place.  
With the new [ASG update]({% post_url 2018-11-18-asg-with-multi-instance-types %}) we can use multiple instance types for the Spot ASG,
CloudWatch Alarms for scaling and another ASG to fallback on OnDemand instances.


## Architecture

We will use [AWS CloudFormation Template][aws-cloudformation-template] to deploy the following:
* [Elastic Load Balancing][aws-elb]
* [LaunchTemplate][aws-launch-template]
* Two [AWS AutoScalingGroup][aws-asg]
* Two [Amazon CloudWatch Alarms][aws-cloudwatch-alarm]
* Two [Scaling Policy][aws-scaling-policy]

The GOAL of the architecture is to run as much Spot capacity as possible and decrease the risk of application impact due to Spot interruption or un-availability. 
If Spot capacity is insufficient, additional OnDemand capacity will be ready to scale out fast.  
We will use two Auto Scaling Groups; the first will be for the Spots (the main capacity supplier) and the second group for the OnDemand instances.
If the Spot group is unable to fulfill the target capacity and scale out due to lack of capacity, there CPU utilization will grow until it will breach the CloudWatch CPU Utilization alarm that will trigger the OnDemand group scaling policy to scale out.  
Once the Spot group will fulfill its target capacity of CPU Utilization of 40% _(The actual CloudWatch metric thresholds are examples, as these vary across the different applications and regions)_ it will be lower then CloudWatch CPU Utilization alarm threshold of CPU Utilization 60% that will scale in the OnDemand group.  
This solution will also work for Spot interruption, as when the Spot is lost, the Auto Scaling Group will try to create another spot from a different type. If it will success - Great, if not - the CPU will grow again and that will trigger that alarm.  
_(Optional: A CloudWatch Rule can be created from the `EC2 Spot Instance Interruption Warning` event to scale out the OnDemand group)_

The following diagram shows how the components work together.

![AutoScalingGroup with OnDemand fallback]({{ site.url }}/assets/images/stop-using-spotinst/spot-ondemand-asg.png){: .image-center }

## Review the details
_All the code and cloudformation templates are minimal and shows only whats needed for this blog post._

_The actual CloudWatch metric thresholds are examples, as these vary across the different applications and regions._


#### Elastic Load Balancing
The ElasticLoadBalancing is just the entry point for traffic in this example, Nothing fancy here.

{% highlight json %}
{
  "ELB": {
    "Type": "AWS::ElasticLoadBalancing::LoadBalancer",
    "Properties": {
      "LoadBalancerName": {"Fn::Sub": "${AWS::StackName}-elb"},
      "HealthCheck": {
        "Target": "HTTP:80/server/healthCheck.php",
        "HealthyThreshold": 3,
        "UnhealthyThreshold": 3,
        "Interval": 20,
        "Timeout": 15
      }
    }
  }
}
{% endhighlight %}

#### Launch Template
The only way to work with multiple instance types in ASG is with Launch Template. It should contain all the configuration to launch an instance.
Note that the `InstanceType` field is not really mandatory. The real instance types will be configured in each ASG.

{% highlight json %}
{
  "LaunchTemplate": {
    "Type": "AWS::EC2::LaunchTemplate",
    "Properties": {
      "LaunchTemplateName": {"Fn::Sub": "${AWS::StackName}-lt"},
      "LaunchTemplateData": {
        "InstanceType": "m4.large",
        "Monitoring": {
          "Enabled": true
        }
      }
    }
  }
}
{% endhighlight %}

#### The Auto Scaling Groups
We will configured 2 Auto Scaling Groups, one for the Spots and the other for the OnDemand.
Pay attention that the `MinSize` for the Spots group is 1 while in the OnDemand group it is 0. This because we want to allow the group to scale down to 0 when no OnDemand instances are needed.

Because we are going to use CloudWatch Alarm based by the Spots group CPU Utilization, I set the `OnDemandBaseCapacity` to 1 so there will be always a CPU utilization metric for the Spots group (in case there will be no spots available at all).

{% highlight json %}
{
  "SpotASG": {
    "Type": "AWS::AutoScaling::AutoScalingGroup",
    "Properties": {
      "AutoScalingGroupName": {"Fn::Sub": "${AWS::StackName}-Spots"},
      "AvailabilityZones": {"Fn::GetAZs": {"Ref": "AWS::Region"}},
      "MinSize": 1,
      "MaxSize": 10,
      "HealthCheckType": "ELB",
      "MixedInstancesPolicy": {
        "InstancesDistribution": {
          "OnDemandBaseCapacity": 1,
          "OnDemandPercentageAboveBaseCapacity": 0,
          "SpotAllocationStrategy": "lowest-price"
        },
        "LaunchTemplate": {
          "LaunchTemplateSpecification": {
            "LaunchTemplateId": {"Ref": "LaunchTemplate"},
            "Version": {"Fn::GetAtt": ["LaunchTemplate", "LatestVersionNumber"]}
          },
          "Overrides": [
            {"InstanceType": "c5.xlarge"}
            {"InstanceType": "c5.2xlarge"},
            {"InstanceType": "c5.4xlarge"},
          ]
        }
      },
      "LoadBalancerNames": [
        {"Ref": "ELB"}
      ]
    }
  },

  "OnDemandASG": {
    "Type": "AWS::AutoScaling::AutoScalingGroup",
    "Properties": {
      "AutoScalingGroupName": {"Fn::Sub": "${AWS::StackName}-OnDemand"},
      "AvailabilityZones": {"Fn::GetAZs": {"Ref": "AWS::Region"}},
      "MinSize": 0,
      "MaxSize": 10,
      "HealthCheckType": "ELB",
      "MixedInstancesPolicy": {
        "InstancesDistribution": {
          "OnDemandPercentageAboveBaseCapacity": 100
        },
        "LaunchTemplate": {
          "LaunchTemplateSpecification": {
          "LaunchTemplateId": {"Ref": "LaunchTemplate"},
          "Version": {"Fn::GetAtt": ["LaunchTemplate", "LatestVersionNumber"]}
          },
          "Overrides": [
            {"InstanceType": "m5.xlarge"}
            {"InstanceType": "m5.2xlarge"},
            {"InstanceType": "m5.4xlarge"},
          ]
        }
      },
      "LoadBalancerNames": [
        {"Ref": "ELB"}
      ]
    }
  }
}
{% endhighlight %}

#### Spots ASG Scaling Policy
This [Target Tracking Scaling Policy](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scaling-target-tracking.html) will hold the group on ~40% CPU Utilization.
Nothing fancy here either.

{% highlight json %}
{
  "TargetScalingPolicy": {
    "Type": "AWS::AutoScaling::ScalingPolicy",
    "Properties": {
      "AutoScalingGroupName": {"Ref": "SpotASG"},
      "AdjustmentType": "ChangeInCapacity",
      "PolicyType": "TargetTrackingScaling",
      "TargetTrackingConfiguration": {
        "TargetValue": 40,
        "PredefinedMetricSpecification": {
          "PredefinedMetricType": "ASGAverageCPUUtilization"
        }
      }
    }
  }
}
{% endhighlight %}

#### OnDemand ASG Scaling
Here is where the magic happens. When there will be no Spots available, the CPU Utilization of the group will grow and grow until it will breach the below alarm threshold (60%).
The alarm will trigger the Scaling Policy that will add 1 instance to the OnDemand group (sometimes it will be better to add 2 instances at once, to help handle the load quickly). The Spots ASG will try to scale in/out the group to reach the 40% CPU utilization, those making the below alarm to OK state that will remove OnDemand instances from the group.

{% highlight json %}
{
  "SpotsCPUAlarmHigh": {
    "Type": "AWS::CloudWatch::Alarm",
    "Properties": {
      "EvaluationPeriods": 3,
      "Statistic": "Average",
      "Threshold": 60,
      "AlarmDescription": "Spots ASG CPU Utilization",
      "Period": 60,
      "AlarmActions": [
        {"Ref": "OnDemandScalingPolicy"}
      ],
      "AlarmActions": [
        {"Ref": "OnDemandScalingPolicy"}
      ],
      "Namespace": "AWS/EC2",
      "Dimensions": [
        {
          "Name": "AutoScalingGroupName",
          "Value": {"Ref": "SpotASG"}
        }
      ],
      "ComparisonOperator": "GreaterThanThreshold",
      "MetricName": "CPUUtilization"
    }
  },

  "OnDemandScaleUpPolicy": {
    "Type": "AWS::AutoScaling::ScalingPolicy",
    "Properties": {
      "AdjustmentType": "ChangeInCapacity",
      "AutoScalingGroupName": {"Ref": "OnDemandASG"},
      "StepAdjustments": [
        {
          "MetricIntervalUpperBound": -10,
          "ScalingAdjustment": -1
        },
        {
          "MetricIntervalLowerBound": -10,
          "MetricIntervalUpperBound": 10,
          "ScalingAdjustment": 0
        },
        {
          "MetricIntervalLowerBound": 10,
          "ScalingAdjustment": 1
        }
      ]
    }
  }
}
{% endhighlight %}

## Test
Of course all we have to do now is test it out;

Setting the Spots ASG to `min = max = 2`, and to lower the CPU Utilization threshold, will help testing the OnDemand ASG scale in/out.
The Spots ASG CPU Utilization will reach the threshold very quickly, there will be no Spots to scale out (because of the `min = max = 2`), those OnDemand instance should be launched.

When this scenario works, the min amd max can be changed to `min = 2, max = 10`. By that, the Spots ASG will have the ability to launch more spots to help with the load, and eventually the CPU Utilization of the group will decrease, those leading OnDemand instance to scale in. 

<br>
***Hope this post helped you at least in some way.  
Dont forget to leave a comment :)***

[aws-cloudformation-template]: https://aws.amazon.com/cloudformation/aws-cloudformation-templates
[spotinst-elastigroup]: https://spotinst.com/products/elastigroup
[aws-asg]: https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html
[aws-cloudwatch-alarm]: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html
[aws-elb]: https://aws.amazon.com/elasticloadbalancing
[aws-launch-template]: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-launch-templates.html
[aws-scaling-policy]: https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scale-based-on-demand.html