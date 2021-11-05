---
title: "What AWS Doesn’t Tell You About Their Managed Kubernetes Service (EKS)"
date: 2021-07-01T08:34:30-04:00
link: https://medium.com/blu-flame-technologies/what-aws-doesnt-tell-you-about-their-managed-kubernetes-service-eks-1d58c6005e72
header:
  teaser: /assets/images/posts/eks_iam/eks.jpg
categories:
- blog
tags:
- devops
- kubernetes
- aws
- cloud
---

So, you want to run Kubernetes in AWS and it looks like Amazon’s Elastic Kubernetes Service is your golden ticket… Not so fast. There are a ton of gotchas. Let’s walk through a few of those gotchas as we provision a private EKS cluster in AWS.

![EKS](/assets/images/posts/eks_iam/eks.jpg)

_<small>Image Credit: [Amazon AWS](https://aws.amazon.com/)</small>_
