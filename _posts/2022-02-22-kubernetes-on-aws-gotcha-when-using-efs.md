---
title: Kubernetes on AWS — gotcha when using EFS
date: 2022-02-22 12:00:00
categories: [kubernetes,aws]
tags: [kubernetes,aws,efs]
---

![Kubernetes on AWS - gotcha when using EFS](/assets/img/2022/02/dont_panic.jpg)

I was happily migrating a lot of applications over to EKS when my pods went into an **Init** state. Panic set in and I was having a look around at what the problem could be (what is it this time, I said to myself).

For this specific Kubernetes cluster I was using a single **EFS** volume for all my deployments into quite a few separate namespaces. The problem I eventually faced was that when you create a persistent volume, it registers as an **Access Point** on an **EFS mount** … and turns out there is a **hard limit of 120 Access Points for each file system**. So when I hit this limit, my new pods failed to provision persistent volumes. PANIC STATIONS!

The way I got around this limitation was to **create additional EFS volumes** and **deploy multiple EFS storage classes in Kubernetes** tied to these new volumes and **balance out the allocation** of new application deployments to those new storage classes. For now that seems to be working stable, but I will keep you posted if there are any changes or updates. The other limit on EFS to keep a keen eye on is **"Number of mount targets for each VPC"** which is **400**.

Have you encountered this issue? What do you think is the best way to solve the problem? Did this post help you in any way? Please feel free to comment below.

Ensure you read and understand the limits imposed on EFS, the hard limits stop you dead in your tracks.

Following are the quotas on Amazon EFS resources for each customer account in an AWS Region:

* Number of access points for each file system = 120
* Number of connections for each file system = 25,000
* Number of mount targets for each file system in an Availability Zone = 1
* Number of mount targets for each VPC = 400
* Number of security groups for each mount target = 5
* Number of tags for each file system = 50
* Number of VPCs for each file system = 1

More here: [https://docs.aws.amazon.com/efs/latest/ug/limits.html](https://docs.aws.amazon.com/efs/latest/ug/limits.html)

Also ensure when you are using Amazon EFS CSI driver that the limits are also imposed here: [https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html](https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html)
