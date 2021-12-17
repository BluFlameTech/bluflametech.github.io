---
title: "Managing Terraform At Scale"
subtitle: Techniques For Managing Terraform State and Modules
date: 2021-11-30T11:50:30-04:00
link: https://levelup.gitconnected.com/managing-terraform-at-scale-d71d6b7a0357
header:
  teaser: /assets/images/posts/recovering_tf/terraform.jpg
categories:
- blog
tags:
- terraform
- infrastructure
- devops
- aws
---

Terraform is a great tool for developing Infrastructure as Code (IaC). But it’s not without a few gotchas, specifically 
the management of Terraform state and reusable Terraform modules. Fortunately, with a little forethought and some effort, 
these challenges can be overcome.

![Terraform](/assets/images/posts/recovering_tf/terraform.jpg)

_<small>[HashiCorp Terraform](https://www.terraform.io/) Logo</small>_
