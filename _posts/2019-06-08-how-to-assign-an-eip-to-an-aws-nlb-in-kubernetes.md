---
title: How to assign an EIP to an AWS NLB in Kubernetes
date: 2019-06-08 12:00:00
categories: [kubernetes, aws]
tags: [kubernetes,aws,eip,nlb]
---

Out of the box Kubernetes has no support for assigning an EIP to an AWS Network Load Balancer. There is currently a pull request for this feature listed here: [https://github.com/kubernetes/kubernetes/issues/63959#issuecomment-484203661](https://github.com/kubernetes/kubernetes/issues/63959#issuecomment-484203661)

This feature request has unfortunately not been approved yet.

Not to worry though I have put together a simple application you can run as a CronJob on your current Kubernetes cluster that will clone the targets from your Kubernetes Service LoadBalancer to an AWS Network Load Balancer that you create and have an EIP assigned to.

Docker hub repository: [https://hub.docker.com/r/newtondev/k8s-aws-nlb-target-cloner](https://hub.docker.com/r/newtondev/k8s-aws-nlb-target-cloner)

Source code repository: [https://github.com/newtondev/k8s-aws-nlb-target-cloner](https://github.com/newtondev/k8s-aws-nlb-target-cloner)

Medium Article: [https://craignewtondev.medium.com/how-to-assign-an-eip-to-an-aws-nlb-in-kubernetes-17c2b5894728](https://craignewtondev.medium.com/how-to-assign-an-eip-to-an-aws-nlb-in-kubernetes-17c2b5894728)