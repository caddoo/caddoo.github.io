---
layout: post
title: Running an instance of Matomo analytics using AWS & GitHub Actions - Part 1 - Defining the infrastructure
date: 2023-07-28
---

## What am I trying to do?

I want to run a PHP application with a database where the infrastructure is versioned in Git & the changes can be pushed up to AWS via GitHub Actions.

## This article

This article is part 1 of 3, this article will be focussing on defining the problem, and then defining and explaining a suitable infrastructure in AWS.

Part 2 will be defining the infrastructure in AWS CDK and bootstrapping the application.

Part 3 will be automating infrastructure changes via GitHub actions.

## Why do I want to do this?

The idea here is to deploy and manage my own instance of an analytics tool for this very blog, I could quite easily just use Google Analytics and include the JavaScript however there are a few concerns out there regarding [Google Analytics and Europe](https://www.wired.com/story/google-analytics-europe-austria-privacy-shield/).

## Why these tools?

### AWS CDK

In the past I have worked with [AWS CloudFormation](https://aws.amazon.com/cloudformation/), I think this is definitely a step-up from manually setting up your infrastructure, however it still didn't feel like the most comfortable thing to work with.

It lacked all the tools you get with a full programming language, while it did have a [linting tool](https://github.com/aws-cloudformation/cfn-lint) that you could set up in your editor and CI that made it a little easier, I still feel like there was a lot of guess work and trial and error leading to a lot of swearing and far too many hours waiting and getting hit with errors in CloudFormation runs in the AWS console.

Along came [AWS CDK](https://aws.amazon.com/cdk/), it's like a layer on top of CloudFormation were you can write your infrastructure as code in many languages (unfortunately not PHP) and it then transpiles itself into cloudformation.

You get the nice auto complete, you pick up some errors on compilation (if a compiled language of course), it's also a familiar language for you and other developers.

You can write unit tests! But also snapshot tests checking the produced CloudFormation output is what you expect.

Overall as a developer this is something I'm much more comfortable working with.

And why AWS? Well there is no real reason here this could easily be done with Azure or GCP, or if I was feeling fancy I could use a cloud-agnostic tool to avoid the vendor lock in.

But one of the big benefits of using these large providers like AWS, Azure or GCP is that it really reduces the development time needed to get a stack online, and adding another layer to make it agnostic will probably add all that time back on. Plus this stack is so simple that transferring it to another cloud provider wouldn't take long at all.

### GitHub Actions

I don't have a strong preference for GitHub Actions, but it is something that I haven't worked that much with in the past, so this is a good opportunity to try it out.

I have mostly worked with TeamCity for CI/CD however I didn't choose this for this little project because GitHub Actions manages the infrastructure that supports the CI/CD platform, whereas with TeamCity you have to manage it yourself.

Because we are already having to manage our infrastructure for our analytics software I think that's enough.

## What is the goal?

I want to host my own instance of a website analytics tool and be in control of the data myself, the tool of choice is [Matomo](https://matomo.org), I have a little bias here as this is the company I've just started working for, so it makes sense to get used to what I'll be working with. But also it's a more privacy focused tracking tool and the data will be stored in my own database which I think my users will be more comfortable with.

Additionally, this will be installed on an EC2 instance with a MySQL RDS instance as well, I will also implement some good security practices locking the application down as much as possible.

In the end this blog will include a piece of JS code from my own EC2 that will be used for analytics for this blog.

## Prerequisites 

- AWS CLI installed
- AWS CDK installed
- Whatever language installed (I'll use C#)
- GitHub account

## Requirements

- Available in one region `eu-west-1`
- The EC2 instance should be stateless, making it possible to spin up new ones easily
- Database should not be accessible externally
- The application should be available from https://analytics.caddoo.net

## Infrastructure

I'm going to attempt to build this, so it's as low cost as possible, while also making it possible to scale if by some miracle the traffic on this blog increases significantly.

If you don't care about quick scalability then you could run most of your application and it's dependencies (DB) on a single EC2 instance which would significantly reduce costs.

Here is a simplified diagram of the infrastructure I'm planning to make, I've excluded some resources just to put focus on what is important.

![AWS diagram of load balancer, EC2 and database](/assets/images/aws-php-appliation-diagram.png)

### Subnet separation
As you can see this is quite a simple setup, I have split the resources into three subnets:

1. Public - Accessible from the outside world, and can also communicate with the internet.
2. Private with Egress - Only accessible from inside the VPC but can communicate with the outside world.
3. Private isolated - Only accessible from inside the VPC, can't communicate outside.

This separation is important from a security point of view, you really only want to give access to the internet for those resources that absolutely require it.

Another possible reason is compliance, certain industries and regulations will require you to have data separate from other data, subnets help you achieve this.

### EFS - Elastic file storage

I've gone for a separate resource to store the files of the PHP application, its good practice to make the EC2 instances stateless, as this allows you to create another one, add it to the load balancer which relative ease.

Imagine if your EC2 instance had state, and the application has created new files, changed some config files for example. If you start to get more traffic your only option is vertically scaling the instance by adding more resources (memory & CPU for example). This approach would come with a little downtime which isn't ideal.

### NLB - Network load balancer

There are a couple of options with AWS regarding the load balancer, a network load balancer or an application load balancer. I've listed the differences below:

    Network load balancer

    - Operates at the transportation layer of the OSI model - Layer 4
    - Protocol agnostic (use things other than HTTP/S)
    - High performance
    - Doesn't mutate requests
    - Health checks services by just making sure ports are accesible
    - Cheaper

    Application load balancer

    - Operates at the application layer of the OSI model - Layer 7
    - Works with HTTP/HTTPS/HTTP/2
    - It can route based on request content
    - Modifies requests before passing them onto to services
    - Can peform health checks using HTTP endpoints
    - Expensive

For the load balancer in this article, there are three requirements:

1. Terminate HTTPS
2. Route HTTPS (443) traffic to HTTP (80) on the EC2
3. For certain IPs route SSH (22) to SSH (22) on the EC2

Based on these requirements we can safely go with the network load balancer with the added bonus of cost saving & performance 🎉.


## Wrapping up

We have now defined the problem, the infrastructure solution, and in the next article we will look at making infrastructure deployable using AWS CDK.

