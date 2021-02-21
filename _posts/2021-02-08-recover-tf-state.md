---
title: "Disaster Recovery for Terraform State"
date: 2021-02-08T11:45:00-05:00
header:
  teaser: /assets/images/posts/recovering_tf/terraform.jpg
categories:
- blog 
tags:
- devops
- terraform
- infrastructure
- disaster recovery
---

Anyone who has worked with Terraform knows the pain associated with managing Terraform state. Things
get even more interesting when factoring in disaster recovery (DR) for Terraform state. If that rings 
true for you, read on to find a relatively simple approach to DR with Terraform state.

![Terraform](/assets/images/posts/recovering_tf/terraform.jpg)

_<small>Credit: HashiCorp Terraform</small>_

### What is Terraform?

Terraform is Infrastructure as Code (IaC) that allows you to [mostly] declaratively define infrastructure.
The idea is that infrastructure definitions can be quickly and easily defined, managed as composable units
and versioned using common source control tools (Git, Subversion, Perforce, etc). In a nutshell, it 
allows you to treat infrastructure similarly to how you might treat other source code.

A common use case for Terraform is cloud-based infrastructure. And to that end, Terraform can manage 
resources in AWS, Azure, Google Cloud and more. Spoiler: the example provided in this post will be in the context 
of AWS.

### Terraform State

Terraform uses something called [state](https://www.terraform.io/docs/language/state/index.html) (commonly in the form of state files) to manage, well, the state
of Terraform configurations. Every Terraform [root module](https://www.terraform.io/docs/language/modules/index.html#the-root-module)
has an associated state, which is representative of the infrastructure following its last 
[apply](https://www.terraform.io/docs/cli/commands/apply.html). And when an apply command is issued
against a root module that has an existing state attached, Terraform first refreshes its state to
reconcile the actual infrastructure state with the root module's last known state. This allows
for configuration drift detection. It also enables Terraform to best determine how to apply configuration changes
that were made since the last apply.

### The Problem With Multiple Regions

Let's say you have a situation where you have resources from multiple 
[regions](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.RegionsAndAvailabilityZones.html) bundled 
together in a single Terraform root module. Let's further assume that when the root module is applied, all
of the regions associated with those resources are 100% available. In this scenario, the Terraform apply works as 
expected: the state is updated and all of the resources are provisioned and wired up just as described in the root
module configuration.

Now, let's say there is a partial region outage in one of the regions where resources were targeted in that root module.
Even assuming that access still exists to the associated state file, any future Terraform apply executed against that root 
module would fail. Can you see why?

Above, we said that on apply, Terraform first tries to refresh its state. If there is an outage that affects one or more
resources in the root module, a state refresh would not be possible and the configuration along with its associated
state would be in a non-working state. 

### An S3 Example

![S3 CRR](/assets/images/posts/recovering_tf/s3_repl.jpg)

_<small>S3 Replication; Created using [Cloudcraft](https://www.cloudcraft.co/)</small>_

A common deployment scenario with S3 is to have a primary [S3 bucket](https://aws.amazon.com/s3/) in one region that 
replicates its data to another bucket in a different region. This is called 
[S3 cross-region replication](https://aws.amazon.com/s3/features/replication/). In this scenario, if there was an 
availability problem with either region, the data would still be available. However, Terraform would not be able 
apply future root module configuration updates.

If the source bucket's region was lost, then you might want to spin up a different bucket in an available region
and switch the replicated bucket to be your new source bucket. Or, if the destination bucket's region was lost, then
you might want to spin up a new bucket in a different region and change the source bucket to replicate to that new
bucket. But... if you provisioned both the source bucket and the destination bucket from the same root module
configuration, then you might be in trouble.

#### Terraform Modules to The Rescue

If the S3 destination bucket were in one root module and the S3 source bucket were in a different root module, then
there wouldn't be a problem since each root module would have its state confined to only one region. But this is, quite
frankly, a pain in the ass. Because now you have to do Terraform apply twice (once for each root module) and you have
to manually sequence the order in which resources in different regions are provisioned - that's supposed to be 
Terraform's job!

What we can do, however, is break the provisioning of each S3 bucket into its own 
[submodule](https://www.terraform.io/docs/language/modules/develop/composition.html) and have a root module
orchestrate the provisioning of both. 

![S3 CRR](/assets/images/posts/recovering_tf/s3_repl_modules.jpg)

_<small>S3 Replication Using Submodules; Credit: Cloudcraft</small>_

This solves the problem of having to manually sequence two Terraform apply operations to effectively accomplish a single
operation, albeit across more than one region. But we still have the problem with state from multiple regions being
combined into a single state file... or do we?

#### To Import or Not to Import?

By breaking down the submodules by region, some level of isolation through abstraction is achieved. It's not quite 
enough to solve the resources across multiple regions stored in a single Terraform state file problem. But it feels like
we are closer. And indeed, we are.

To finish the solution, it's necessary to understand that the only major differences between a Terraform submodule and a
Terraform root module are that a Terraform root module has state attached to it and Terraform commands can be directly 
executed against a Terraform root module. Of course, there are other subtle differences. But in most cases, there is 
nothing stopping us from using a Terraform submodule as a Terraform root module.

Fortunately, Terraform provides a mechanism to do exactly that: 
[Terraform import](https://www.terraform.io/docs/cli/import/index.html). Terraform import allows Terraform to bring
existing infrastructure under the management of new Terraform state. Using Terraform import, we could initialize 
one (or both) of the S3 submodules as a root module and then import the existing corresponding S3 bucket, which would then allow 
the s3 submodule to manage the S3 bucket independently of the other S3 bucket (that resides in a different region).

#### What About More Robust Configurations?

The problem with import is that it must be executed on a per-resource basis. This is OK for the example above since
each module only deals with a single resource. But what about more robust configurations that contain more than one
resource per module? That's where [Terraform state mv](https://www.terraform.io/docs/cli/commands/state/mv.html) comes 
into play. 

[Terraform state mv allows submodule state to be extracted into its own state](https://www.terraform.io/docs/cli/commands/state/mv.html#example-move-a-module-to-another-state).

### Summary

* Availability outages can cause trouble for Terraform configurations - this must be a consideration when designing Terraform configurations.
* Break down Terraform configuration region dependencies into submodules.
* Orchestrate submodules using a root module.
* During an outage, Terraform import or Terraform state mv can be used to allow submodules to manage the state of their resources.