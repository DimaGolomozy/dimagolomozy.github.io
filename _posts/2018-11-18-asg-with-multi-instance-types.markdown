---
layout: post
title:  "AWS AutoScalingGroup Update - Now With Multiple Instance Types"
description: "Review and Comparison between Amazon ASG update that now supports multiple instance types with Spot Fleet"
image: assets/images/asg-update/asg-multipe-types.png
comments: true
date:   2018-11-18
tags: [AWS, ASG, EC2, Spot Fleet]
---
On 14 Nov 2018, Amazon released an [update][aws-asg-update] for [Auto Scaling Groups][aws-asg] with new features that we all were waiting for a long time!
* Multiple Instance Types
* New Pricing Models

The one that is the most significant in my opinion is the **_Multiple Instance Types!!!_**  
For a long time we had to use a **single** instance type for our Auto Scaling Group, and if we wanted to use a combination of types we had to use the [Spot Fleet][aws-spotfleet] service _(and we all know that it's not the same)_, and sometimes even creating multiple Auto Scaling Groups for each instance type.  
Now, we have the ability the choose a combination of instance types and pricing models in a single Auto Scaling Group that works just like the Spot Fleet but much better.


#### Auto Scaling Group Vs. Spot Fleet  

|                             |          Auto Scaling Group        |     Spot Fleet    |
|-----------------------------|:----------------------------------:|:-----------------:|
| **Multiple Instance Types** |                  V                 |         X         |
| **Health Check**            |  EC2 State Check, ELB Health Check |  EC2 State Check  |
| **On Demand Usage**         |           Percentage of OD         | Fixed OD Capacity |
| **Edits**                   |                  V                 | Immutable Request |
| **Update Policy**           |  Rolling Update, Replacing Update  |         X         |
| **Termination Policy**      |                  V                 |         X         |
| **Instance Weights**        |                  X                 |         V         |
| **TTL**                     |                  X                 |         V         |
| **Scaling**                 |                  V                 |         V         |
| **ELB/ALB association**     |                  V                 |         V         |
| **Lifecycle Hooks**         |                  V                 |         X         |
| **Notifications**           |                  V                 |         X         |  

Lots of [Lambda][aws-lambda] functions were necessary for Spot Fleet to handle some basic things; Updates, Health Check from ELB, Adding Target Group, etc...
Now, one can move his Spot Fleet to Auto Scaling Group and enjoy better integration with all the features above.

Enjoy scaling...


[aws-asg-update]: https://aws.amazon.com/blogs/aws/new-ec2-auto-scaling-groups-with-multiple-instance-types-purchase-options
[aws-asg]: https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html
[aws-spotfleet]: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-fleet.html
[aws-lambda]: https://aws.amazon.com/lambda
