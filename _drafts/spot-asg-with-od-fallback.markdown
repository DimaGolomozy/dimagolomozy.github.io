---
layout: post
title:  "Going Back From Spotinst to AWS AutoScalingGroup"
description: "Use AWS AutoScalingGroup with Spots and OnDemand for fallback and not Spotinst Elastigroup"
image: assets/images/folder/image.png
comments: true
date: 2019-01-10 14:00:00
tags: [AWS, Spotinst, Elastigroup, ASG, Spots, OnDemand, OD, Fallback, CloudFormation]
---

[Spotinst Elastigroup][spotinst-elastigroup] bring lots of nice features to the table. They have the ability to define a "single" group with multiple instance types, Spots and OnDemand, and can predict Spot behavior, capacity trends, pricing, and balance the capacity accordingly.

BUT, what if we don't need all those fancy features, and just want ASG with Spots and OnDemand for fallback (and to lower the "cloud" costs) - well this post is the place.

I'll start by saying that this post will be about how to create ASG of Spots with OnDemand fallback
For a long time we couldn't use multiple instance types in [AWS ASG][aws-asg] (From 14 Nov 2018 we can - read more about it [here]({% post_url 2018-11-18-asg-with-multi-instance-types %})),
nor could we use Spots with combination of OnDemand instances in a single ASG.














[spotinst-elastigroup]: https://spotinst.com/products/elastigroup
[aws-asg]: https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html
[aws-lambda]: https://aws.amazon.com/lambda
