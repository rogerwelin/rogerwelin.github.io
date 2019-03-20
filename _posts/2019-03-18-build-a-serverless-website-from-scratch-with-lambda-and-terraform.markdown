---
layout: post
comments: true
title:  "Build a serverless website from scratch using S3, API Gateway, AWS Lambda, Go and Terraform"
date:   2019-03-18 19:43:43
categories: aws serverless terraform lambda
---

In this guide we will leverage AWS to build a completely serverless website (frontend and backend api) using S3, API Gateway and Lambda.
We will use Terraform to set up our AWS resources and we'll use Go as our programming language for the lambda function. For the frontend we will write a static html and javascript/jquery file (stored in S3) that will communicate with our lambda function. 

The goal of this guide is to show how to leverage infrastructure-as-code, i.e. no clicking and fiddling around in the AWS console!

![weather-website](/assets/images/lambda2.jpg)

<!-- more -->

### **Disclaimer**
By following this tutorial you will create resources in your AWS account that will cost money, I strongly recommend that you remove them after reading this guide. This guide does not claim to follow best practices, API Gateway will be used in a quite basic manner; we will proxy all request to our Lambda that will have to manage the routing logic. 
The choice of using Terraform for setting up the AWS resources was based solely on personal preference, for example the [serverless framework](https://serverless.com/) and [apex up](https://up.docs.apex.sh/) are tools that can accomplish the same thing.


### **The Website**
The website that we'll build will be a web app where users can get the weather information for different cities. The static website will make a request to our Lambda that will query an external weather api service, mangle the json and return the data back to the client. It will look something like this when all the pieces are assembled:

![weather-website](/assets/images/weather.gif)


### **Preparation**

For this tutorial you will need the following "dependencies":

* AWS account & AWS credentials (required)
* Terraform (required)
* aws-cli (optional but recommended)
* Go (optional but recommended)
* Clone accompanied github repo (required)

**AWS Credentials**  
Terraform (and aws-cli) needs to access your AWS credentials to be able to create resources. We'll need a user with at least permission to Api Gateway, S3, Lambda and Cloudwatch with programmatic access. After creating the user copy the **API Key** and **Secret** to a file in the path *$HOME/.aws/credentials*. If unsure how to generate the file you can use awscli:

````bash
$ aws configure
```

**Terraform**  
To install Terraform head over to the downloads page and pick the package for your OS: https://www.terraform.io/downloads.html

**aws-cli**  
To install awscli use the following command:

````bash
$ pip install awscli --user
```

**Go**  
To install Go consult your distro's package manager or head over to the download page: https://golang.org/dl/

**Clone Github Repo**  
The following repo contains the frontend code, Go lambda code and Terraform code used in this guide, get it by running the following command:

````bash
$ git clone https://github.com/rogerwelin/serverless-website-tutorial
```

### **Setting up S3 Buckets**

We are going to set up two S3 buckets, one to store the lambda artifact (zip-file) and one that will be the actual website. Navigate to the folder *terraform/s3*.
Inspect the *vars.tf* file, here we define variables that will be used in the main module:

{% gist f310c619cdf643bd63e7d102ebd0e86f %}

Here I have chosen to spin up resources in the us-east-1 region in AWS (change this value based on your prefered region). Also S3 bucket names must be unique since its a global service, hence for the *website_bucket* and the *artifact_bucket* variables you might have to test a few names since many S3 bucket names are taken.

The *main.tf* file are not terrible exciting since it just basically creates two buckets (one public and one private):

````bash
$ terraform init
$ terraform apply -auto-approve
```

If everything went well; good job, we are ready to move on! If not examine the error message outputted by Terraform. Most likely it will say that the bucket name is already taken, in that case just change the name again in the *vars.tf* file and retry.


### **Examine the Go function**

Writing a Lambda function in Go is quite similar to writing a normal http server, however there are some gothas; instead of net/http or mux/gorilla we need a lambda package. This makes local development difficult without hacks. With that being said, the flow of this function is quite straight forward; client POSTs data to the function where the payload is a variable called *woeid* (world id), we'll unmarshal the body and make an external http call to the actual weather api site with the woeid as the resource, we'll mangle the data a bit to make it readable for our frontend and lastly we send the data back to the client. 

Lets take a look at the code below (or navigate to the *faas/* folder), important sections are commented:

{% gist 1bcc1a5e9a894d592d3113970de0cea7 %}

**Compiling the project**    
We will now comple the function, compress it to a zip file (zip is the format Lambda expects) and upload it to our S3 artifact bucket created in previous step:

````bash
$ go get github.com/aws/aws-lambda-go/lambda
$ GOOS=linux go build -o faas main.go
$ zip faas.zip faas
$ aws s3 cp faas.zip s3://{my-artifact-s3-bucket-url}/
```

### **Spin up Lambda & API Gateway**
Now for the fun part; creating the API Gateway and Lambda! Navigate to the *terraform/lambda-apigw-iam* directory. Lets take a look at *vars.tf* first:

{% gist 3b7e4a0666d27854ec2c87d301dd2fd3 %}

If you stuck to us-east-1 when creating the S3 buckets no need to change, otherwise change to the region you picked earlier. Regarding the variables *artifact_zip_name* and *faas_name*; if you followed my name suggestion earlier when compiling the Go function no need to change the values, however if you picked another name change accordingly. For the *artifact_bucket* variable, you need to change to the bucket name you created earlier.


Now lets look at *main.tf*! This module takes care of adding permissions between AWS services, configures and deploys the API Gateway, creates the Lambda function by taking the artifact faas.zip from our S3 bucket and enabled Cloudwatch log permissions. It might look a bit scary when first viewing it but don't freak out! I will try to break it down by taking things piece by piece.

**IAM and permissions**  
Here we create a IAM role (*aws_iam_role*) which we will use for the lambda function. The IAM role dictates what access it has to other AWS services. We want Lambda to access S3 and Cloudwatch (for writing logs) so in *aws_iam_role_policy* we define a policy document which gives Lambda S3 and Cloudwatch access. 
{% gist 11981df2f7fb7817c1909069dde9047d %}


**Create the Lambda function**  
This section is quite straight forward. Here we tell Terraform where to fetch our artifact zip, what the name of the binary is and what our chosen lambda runtime is. In the *aws_lambda_permission* resource we allow API Gateway to invoke our lambda function. The Lambda function will be public (open to the whole world) since our frontend will run in the browser (i.e. no server-side code)


{% gist 2d020d822f803039e055324b706c2006 %}


**Configuring API Gateway**  
Personally I think that API Gateway is a bit non-intuative to work with, this triggered me to write it as Terraform configuration. However as you might see below it's a lot of config! What makes things more complex are that we need to add CORS configuration, meaning a separate OPTIONS method and accompanied headers. The Terraform docs was not super clear how to achieve this either so this was a bit of trial-and-error to get to a working state. 

We are creating an endpoint called */hello* which has two methods *ANY* (frontend will use the POST method) and OPTIONS which is required in order to enable CORS. This [AWS doc](https://docs.aws.amazon.com/apigateway/latest/developerguide/how-to-cors.html) explains quite well CORS and API Gateway. After we have set up all the CORS stuff we can finally set up the ANY method that we will link to our lambda function.

{% gist bd42ff2607ffb075ebca0911b9167e20 %}


Now we are ready to terraform:

````bash`
$ terraform init
$ terraform apply -auto-approve
```

If everything went well Terraform will output the base url of our Lambda function, append the /hello endpoint and let's give it a go with curl:

````bash
$ curl -XPOST -d '{"woeid": "44418"}' https://{api-gw-base-url}/hello
{"weather_state_name":"showers","min_temp":"2.54","max_temp":"14.475","title":"London","lattlong":"52.883560,-1.974060"}
```

Success!! However we are not finished yet, the last thing to do now is to deploy our frontend!

### **Deploy the Frontend**
Navigate to the *frontend/* directory, it contains a file called *index.html*, lets take a look at it:

The html part is pretty basic (due to my lack of frontend skills), it contains a form with a select tag, upon clicking submit button a jquery function will be triggered. **Edit the variable *url* at line 24 with your Api Gateway url** (don't forget to add the /hello endpoint). The jQuery will make a POST to our Lambda function and upon receiving a successful response it will update the paragraph id's in the html body.

{% gist 6677dfa0b09480febe4e2bc5e3d07fe0 %}


To deploy the frontend the only thing we have to do is to upload the index.html file to our website bucket. You can either manually upload it from the AWS console or do it with aws-cli as shown below:

````bash
$ aws s3 cp index.html s3://{your-s3-website-bucket}/ --acl public-read
```

Navigate to https://s3.amazonaws.com/{your-bucket-name}/index.html in the browser and give it a go. If you got it this far; congratulations! We have created a simple responsive website and backend api without virutal machines or web servers to operate and manage, that's pretty cool!

Lets destroy the AWS resources we created in this guide, you can do it with Terraform:

````bash
$ cd terraform/lambda-apigw-iam
$ terraform destroy -auto-approve
$ cd ../s3
$ terraform destroy -auto-approve
```

### **Conclusion**
Serverless is awesome and the opportunities are endless in what we can build with it. Hopefully this guide has given you taste of that. However there are a few things we can improve from this set up which I will cover in another guide.


