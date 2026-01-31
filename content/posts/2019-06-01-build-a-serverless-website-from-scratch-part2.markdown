---
layout: post
comments: true
title:  "Build a serverless website from scratch using S3, Cloudfront, AWS Lambda, Go and Terraform - Part 2"
draft: true
categories: aws serverless terraform lambda cloudfront
---

Welcome to part 2 of how to build a serverless website from scratch using AWS, Terraform, S3 and Lambda. This guide assumes you have read part 1 as we will further build and improve on that solution. If you have not read the first part I'll leave a link [here](https://rogerwelin.github.io/aws/serverless/terraform/lambda/2019/03/18/build-a-serverless-website-from-scratch-with-lambda-and-terraform.html), head over and read it and come back here afterwards.

![weather-website](/assets/images/s3cf.png)

<!-- more -->

### **Improving the System Design**  
The imaginative scenario for our weather site is as follows: our website has gained an unprecedented amount of popularity all over the internet and the traffic has spiked to millions of users each day. The architecture from part 1 seemed fine for a small hobby site but with the amount of traffic the design clearly needs improvement to handle the traffic spike. Each time a user request weather information will trigger a lambda execution which queries the remote weather api. With millions of users each day, the AWS bill racks up (as AWS pays per lambda execution, time and memory). Another aspect is that the remote weather api service will probably blacklist us as we are basically spamming them with requests. 

So we need to rethink how we design this sytem:  
* Do we really need to execute a lambda for each time a user wants to view the weather status? In this case I would say no. Weather status is hardly a realtime problem. If we cache the data for, let's say, 20-30 minutes our users would hardly notice. Thus we can easily "cache" the results
* How to do the caching? API Gateway are able to cache the results for an endpoint for a specified amount of time. This sounds promising, we can thus reduce the load on our lambda and the remote data source. We could also put Cloudfront as a CDN in front and integrate it to the API Gateway
* Can we do better? Instead of caching in API Gateway we can put the entire workload on the client; meaning a totally static site. This has a lot of benefits; less load on our cloud resources and we can remove the complex API Gateway layer.
* How do we improve response times for our global users? As our website bucket is located in the US, users from Asia and Europe will experience slow response times (due to higher RTT, TCP/TLS setup). We should definitely improve this. To accomplish this we should use Cloudfront as a CDN to cache our entire website. Since we no longer have any dynamic content Cloudfront can cache the whole site with data close to the user. When we get new data we can just invalidate the Cloudfront cache

### The new Design  
To quickly re-cap what we will refactor and implement:
* We will scrap API Gateway and re-write our lambda to run at a schedule (cron) using Cloudwatch Events. It will assemble a json file with the current weather data. The json file will be stored in the website S3 bucket. We will thus put all work on the client and since we don't need to talk to the server-side anymore will boost performance
* We will have to rewrite parts of our frontend as we no longer need to invoke a lambda
* We will setup Cloudfront to cache our entire website to serve content as close as possible to the user

### Preparation  

As with part 1 we will clone a github repo containing all code and terraform config, this time also checkout a branch where the above changes are implemented.

```bash
$ git clone https://github.com/rogerwelin/serverless-website-tutorial
$ git checkout -t remotes/origin/improvements
```

### Setting up S3 Buckets  
Nothing has changed since the previous part, we still have one bucket that will be the website and one to store the lambda code as a zip-file. Just run terraform apply to provision the buckets.

```bash
$ cd terraform/s3/
$ terraform init
$ terraform apply -auto-approve
```

### Examine the Go Function

Much of the Go code is the same. The changes I made are removing API Gateway specific code sections as we no longer trigger the lambda from API Gateway. We also need to bring in depencencies to talk to S3. This lambda will be run as a batch. For simplicity sake I hardcoded the world location IDs we want, our lambda will fetch the data from the remote weather api site, tidy up the data and lastly write the content to a json file and transfer it to our website bucket.

Lets take a look at the code below (or navigate to the faas/ folder), important sections are commented:

{{< gist rogerwelin 4ec6d4f59390fdc32d67f492e5a9e530 >}}

**Compile the Go function**  
The Go program uses go mod (which will take care of all the aws dependencies) so we only need to build it and upload to our S3 bucket:

```bash
$ GOOS=linux go build -o faas
$ zip faas.zip faas
$ aws s3 cp faas.zip s3://{my-artifact-s3-bucket-url}/
```
