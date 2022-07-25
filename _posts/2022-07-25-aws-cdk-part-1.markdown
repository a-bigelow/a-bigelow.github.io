---
layout: post
title:  "AWS CDK Part 1 - The Value of Learning By Doing"
date:   2022-07-11 9:00:00 -0400
categories: aws-cdk iac growth technical
---
{% highlight python %}
This blog is the first in a series of posts about AWS CDK, a tool I have many opinions and thoughts about. It is not a how-to guide, but my hope is that you might resonate with the learning path I took and potentially see areas for improvement for either yourself or me. Feel free to reach out in slack @ cdk.dev either way!
{% endhighlight %}

A really obnoxious thing I like to say to people is, "Personal growth lies somewhere between what I'm comfortable doing and what I'm not capable of doing." It's my circuitous way of saying "get out of your comfort zone, it works!" and to be fair, it does.

At one point I was contacted by a Biometrics company to help with a short-term automation project. They had a distributed team of engineers working on an IoT metrics aggregation application, which was built using dotnet and Docker Compose. 

Someone on the team had put together a _mostly_ automated way of performing a build and subsequent deploy of the docker images into AWS. It was an imposing yet functional shell script that demanded a few rounds of errors before you'd get your deploys working -- mainly because bash doesn't come with a built in system to manage dependencies used in a script, ala `npm`. (I do have beef with `node_modules`, but it's vastly superior to having _nothing_.)

What they ultimately wanted was a system where feature-branch code could be checked into a specific git branch, and when a "build" substring was supplied in the commit message, it would spawn a new sample build in their dev AWS account.

What I ended up delivering was a feature-branch-driven pipeline that would perform a `cdk deploy` of a new CloudFormation stack, resulting ultimately in a new instance of the appliance running in AWS that the team could log into and validate as needed. I was able to take advantage of things like [the S3 Assets module](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_s3_assets-readme.html) to automatically load source code into S3, and then pull it down to the build host to run `docker compose up`. Truly, CDK abstracted away a lot of the devops glue that I expected to have to implement myself.

I want to _very_ heavily emphasize that I would never build a system like this again -- the container images should have been built separately and pushed elsewhere, and then securely pulled down to the host before `docker-compose` could be run. If I could go back, I would probably investigate AWS ECS a bit more, I would most definitely decouple the CI and CD systems of my eventual solution, and I almost assuredly would not have chosen Python as my CDK tool of choice (more on that statement later). The one thing I absolutely would not change is the flavor of IaC I decided to learn for this project; AWS CDK.

As someone with more of an Operations background, I had seen AWS CDK in the wild before -- generally when trying to figure out how to modify an AWS Solutions Construct e.g. [the AWS Instance Scheduler](https://github.com/aws-solutions/aws-instance-scheduler/blob/master/source/lib/aws-instance-scheduler-stack.ts). At the time I would take one look at a CDK project, simply feel overwhelmed, and say "Nope!" as I closed the tab. It was simply too much information to try and parse through for a non-developer who had little vested interest in his user story. (Sorry, previous management.)

I have a love/hate relationship with YAML (mostly hate), and just generally don't enjoy writing things that look more like config than automation. I figured my options were Ansible (no), Terraform, Pulumi, and CDK. Mainly due to its proximity to CloudFormation, I chose CDK.

## The Honeymoon Phase

I'm a sucker for good-looking CLI tools. I won't care if your operations take a long time if your loading icon is cool, and/or you throw fun colors in my face. Red is bad and green is rad? I'm in.

I started with the Python tutorials on [cdkworkshop.com](https://cdkworkshop.com). The workshop takes you through a multi-step process of deploying a serverless website with CDK that includes a hit-counter (ostensibly, I didn't finish the workshop). I've never done well with prescriptive learning, so I quickly got impatient and went to go start POCing stuff for my project.

On night #1 I figured I just wanted to get a basic VPC setup. In CloudFormation-land that's a decent amount of work for an evening before dinner, since everything needs to be strung together manually.

After a quick glance at the [Python CDK docs for VPCs](https://docs.aws.amazon.com/cdk/api/v1/python/aws_cdk.aws_ec2/Vpc.html), I'd thought there'd been a mistake in some auto-generated documentation, since this was all I saw at first:

{% highlight python %}
import aws_cdk.aws_ec2 as ec2

vpc = ec2.Vpc(self, "Vpc",
    cidr="10.0.0.0/16"
)
{% endhighlight %}

"Surely there's more", I audibly said during this dramatic retelling. Alas, nope! There really wasn't. If you put that code into your CDK stack, you'll deploy a very bog-standard 3-tier VPC in 3 availability zones. Routing, subnet division, route table associations, and the provisioning of Internet Gateways and NAT Gateways is all taken care of -- just be mindful of the cost of NAT gateways if you leave one of these "running".

This is the simplest example of a CDK VPC -- you can provide any level of config, up to and including a 1:1 CloudFormation config using L1 constructs, to get precisely the VPC you want. For quick POC work, however, this simple implementation was perfect (minus the NAT gateway charges).

This was my first interaction with CDK -- expecting to spend a couple hours on something, and instead spending minutes. This is idiomatic of the majority of my work with the CDK.

## The Difficulties

I maintain that learning CDK is one of the force-multipliers I've stumbled into in my career. Not simply because it enables me to deliver software incredibly quickly from _nothing_, but because of the conceptual challenges I had to overcome when I started dabbling with it.

AWS CDK is built on top of a couple really cool technologies from AWS, namely [Constructs](https://github.com/aws/constructs) (composable pieces of state) and [JSII](https://github.com/aws/jsii), which can be most simply described as an adapter that allows code in other runtimes (i.e. Python, Golang, Dotnet, Java) to interact with Javascript classes. JSII in particular made my head spin a bit when I started with CDK, primarily because I didn't know what it was. My first foray into JSII documentation came after the umpteenth time I said "WHAT ARE ALL THESE `JSII` ERRORS??"

JSII relies on the concept of Protocols to allow users to instantiate objects in multiple runtimes without needing to manually write and release them in multiple runtimes. To the seasoned dev who's familiar with type safety and (to an extent) polymorphism, you're probably already thinking of Typescript interfaces, Golang structs, that sort of thing. Given a predictable shape (or protocol, I'm conflating the two here for ease of explanation), you know you have access to certain properties and can act accordingly. Shapes can be supersets of smaller shapes, or represent a subset of properties from a larger shape. They're a natural key component of a system that is effectively a JSON synthesizer that relies on prescriptive building blocks. (Apologies to anyone in the CDK world that I have offended with that statement.)

To an Ops guy who had used Python primarily for scripting before jumping into CDK, the concept of protocols or "shapes" of objects was entirely foreign to me. When you want an instance of a _class_, you instantiate it overtly -- when you need an object that `implements` a shape, well, that's less overt.

It took me a little while to understand that there are both `interfaces` and `classes` in the CDK docs. I didn't understand the difference between `IVpc` and `Vpc`, for example. 

Writing CDK in Python abstracts this concept away even further. In Typescript, a `SubnetSelection` looks like this:

{% highlight typescript %}
{subnetType: SubnetType.PUBLIC, onePerAz: true}
{% endhighlight %}

It's just an interface, so there's no class to call. Meanwhile, the Python implementation is this:

{% highlight python %}
SubnetSelection(
    subnet_type=ec2.SubnetType.PUBLIC,
    one_per_az=True
)
{% endhighlight %}

Python doesn't have a native way to tell JSII that a `dict` object matches the shape of an `interface`. The closest thing you get, if you're defining a construct in Python, is a decorator that tells JSII which `interface` you're attempting to match, (think the `implements` keyword in Typescript):

{% highlight python %}
import jsii
from jsii_dependency import IJsiiInterface

@jsii.implements(IJsiiInterface)
class MyNewClass():
{% endhighlight %}

I say all of this now after having learned a lot more about types, interfaces, polymorphism, etc. I don't imagine that I would have found a better reason to learn about these concepts or use them in my day job without CDK demanding it of me, so really I have CDK to thank for forcing me to learn and grow. 

While learning Typescript at the same time probably would have taken even longer, I think it would have been a gentler introduction to these concepts, given that I would have been using them in a native environment rather than their bastardized, "Pythonic" implementation.

## The Aftermath

So where did this all lead? The project was a success, and drove a lot of value for the team that I built the system for. Their iteration times were cut down by a lot, to be sure. I had also built a _separate_ CDK pipeline to stand the application pipeline back up if it ever fell over, so we even successfully navigated an accidental stack deletion together with minimal issues. (I know now that this could have simply been prevented with a stack policy preventing deletion, but the opportunity to flex my automation was pretty nice.)

More importantly, I started using CDK for _everything_. I learned Typescript, began writing constructs, learned much more about type-safety, protocols, polymorphism, inheritance, you name it. Infrastructure code was helping me be a better developer after years of writing python scripts and not having much of a reason to improve other than time-complexity.

In the next AWS-CDK related post, I intend to talk about "Infrastructure as Software", and how defining infrastructure/application patterns in CDK is helping devops teams learn better delivery methods.