---
title: "Adding Capacity Matching to Terraform ASG Updates"
date: 2021-03-21T14:17:00-04:00
header:
  teaser: /assets/images/posts/zero_downtime_asg/AWS-autoscaling-logo.jpg
categories:
- blog 
tags:
- devops
- terraform
- infrastructure
- automation
- aws
- solutions
---

A month ago, we discussed [zero-downtime updates for autoscaling groups using Terraform](/blog/asg-zero-downtime-tf/).
But we omitted capacity matching. This is the post that discusses how to do capacity matching.

![AWS Autocaling Groups](/assets/images/posts/zero_downtime_asg/AWS-autoscaling-logo.jpg)

_<small>Image Credit: [Amazon AWS](https://aws.amazon.com/)</small>_

Let's say you provisioned an autoscaling group of web application servers attached to a load balancer with a desired capacity of x.
Now, it's time to push an update that will replace that autoscaling group with a new autoscaling group. But increased
load has caused your autoscaling group to scale up to a capacity of y (y > x).  How can you ensure that the new autoscaling group can
meet the same demand as the autoscaling group that is now being replaced without jumping through manual hoops and without
waiting for the new autoscaling group to gradually scale up? Answer: capacity matching.

### What We Want to Do In a Nutshell

1. Get the capacity of the autoscaling group that is being replaced
2. Spin up a new autoscaling group with the same capacity as the autoscaling group that is being replaced
3. Attach the new autoscaling group to the load balancer
4. Destroy the old autoscaling group once the new autoscaling group is "InService"

Steps 3 and 4 were already covered in [Zero-Downtime AWS Autoscaling Groups with Terraform](/blog/asg-zero-downtime-tf/).
Steps 1 and 2 are new. Below we'll talk about how we achieve them.

### Getting the Capacity of the Autoscaling Group

The number of instances available for service are equivalent to the number of instances attached to the load balancer.
All the requests go through the load balancer. So, the load balancer seems like a reasonable place to look to determine the current 
capacity. The problem is that Terraform doesn't really have a data source that can give us the current capacity in terms of
instances attached to the load balancer. If we want that data, we'll have to get it ourselves.

### An External Data Source

Fortunately, Terraform allows us to provide our own [external data source](https://registry.terraform.io/providers/hashicorp/external/latest/docs/data-sources/data_source).
With an external data source, we can write say, a Python script that can accept input from Terraform and respond with
data that can be consumed within a Terraform configuration. Not bad.

Once we have input data from Terraform, we can use that to determine the current capacity in terms of instances attached
to the load balancer. We also want to pass in a default value in case the load balancer doesn't exist since data sources
have a bad habit of blowing up if what the data source is querying cannot be found (and we don't want that).

To do this, we will need 3 things for input from Terraform:

1. The AWS region
2. The load balancer name
3. The default capacity (in the case where the load balancer doesn't exist)

### The Python Script

Let's first write a Python script in terms of a generic function. Our function will accept a query in the form of a dict and
it will respond with capacity as output in the form of a dict. That's not specific to Terraform. But using dicts as the shape
of both the input and the output will make it easy to integrate with Terraform since Terraform expects both input
and output to be JSON strings.

```python
@terraform
def get_elb_capacity(query):
"""
using the terraform_external_data decorator, get_elb_capacity accepts a dict of input values from Terraform,
validates them and returns the specified load balancer's capacity, if available.
otherwise, the default value supplied is returned.
:param query: a dict of input values from Terraform
:return: a dict containing the load balancer's current capacity
"""

# pre-condition validation
if len(query) != 3:
    raise Exception('incorrect number of input values!')
for key in query:
    if key not in [REGION, LB_NAME, DEFAULT]:
        raise Exception(key + ' is not a valid input value key!')

# query aws loadbalancer by name
# Boto elb reference: https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/elb.html#ElasticLoadBalancing.Client.describe_instance_health
config = Config(
    region_name=query[REGION],
    signature_version='v4',
    retries={
        'max_attempts': 10,
        'mode': 'standard'
    }
)
client = boto3.client('elb')

try:
    load_balancer = client.describe_instance_health(
        LoadBalancerName=query[LB_NAME],
    )
    return {'capacity': str(len(load_balancer['InstanceStates']))}
except botocore.exceptions.ClientError:
    pass
return {'capacity': str(query[DEFAULT])}
```

As you can see, there were a few variables being used that haven't been defined (but you can probably guess what they are).
And you can also see that there's a decorator on top of the function. The decorator isn't strictly necessary. But it does
allow us to separate the functionality from the input and output bindings. Here's what it looks like.

```python
def terraform(func):
  def wrapper():
      tf_input = ''
      for line in sys.stdin:
          tf_input += line
      print(json.dumps(func(json.loads(tf_input))))
  return wrapper
```

The complete script can be found here: [get-elb-capacity.py](https://github.com/BluFlameTech/examples/blob/main/terraform/zero-dt-asg-matching/.scripts/get-elb-capacity.py)

### Updating the Terraform Configuration

The last thing that needs to be done before we can claim capacity matching is to update the Terraform configuration.
First, we must add our new external data source as follows.

```hcl
data "external" "elb" {
  program = ["python3", "${path.module}/.scripts/get-elb-capacity.py"]

  query = {
    region = var.region
    name = "${local.unique_app_name}-lb"
    default = length(var.desired_capacity) > 0 ? var.desired_capacity : var.min_size
  }
}
```

The _region_ is the same region that the rest of the configuration is using. We must pass it in as a parameter, however,
since it's used by the AWS boto library to fetch the load balancer information inside our Python script. The _name_ must
match the load balancer's name exactly. And, finally, the _default_ capacity is the capacity value if no load balacner
exists, which in our case is the stated desired capacity or the minmum capacity size (if the desired capacity isn't specified).

Lastly, the autoscaling group resource must be updated to use the capacity from our external data source as its desired 
capacity.

```hcl
module "asg" {
  depends_on = [
    aws_s3_bucket_object.tar]
  source = "./web_asg"
  app_name = local.unique_app_name
  vpc_id = var.vpc_id
  lb = module.elb
  max_size = var.max_size
  min_size = var.min_size
  desired_capacity = data.external.elb.result.capacity
  ...
}
```

The complete working example is available here: [zero-dt-asg-matching](https://github.com/BluFlameTech/examples/tree/main/terraform/zero-dt-asg-matching)

Happy [infrastructure] coding! Be sure to check back for a future post on how we can test all this stuff.
