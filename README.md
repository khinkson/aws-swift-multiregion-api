# AWS Swift Multi-Region API
[<img src="http://img.shields.io/badge/swift-5.7-brightgreen.svg" alt="Swift 5.7" />](https://swift.org)

Using Swift in the AWS Cloud can take many forms. Here we will focus on a serverless, multi-regional setup. The aim here is to demonstrate how the infrastructure elements fit together to make a distributed API. You will come away from this with a working serverless, multi-regional Swift based API served by [AWS Lambda](https://aws.amazon.com/lambda/).

This project is part of a series that includes other useful elements. For more information see the [Swift in the Cloud Series](https://www.flue.cloud/swift-server-cloud/).

## Planned Infrastructure
With the templates provided, you can create and extend your API to be as close to your users as possible by installing infrastructure in each region of your choosing. In real time, user requests will be routed to the region with the lowest latency from their location.

You are **not limited** to two (2) regions. You can add as many regions as support the underlying infrastructure as you require. __NB__: so far Jakarta, Hyderabad and Osaka do not support the API Gateway V2 HTTP APIs used in these templates.

<img src="https://www.flue.cloud/images/diagrams/aws-swift-multiregion-api.png" alt="Serverless Multi-region API" />

## Requirements
To complete this setup you will need:
1. An AWS account with administrative user access
2. An unused domain name to setup with Route53
3. Xcode 14+
4. Docker 4+
5. AWS CLI 

## Costs & Service Dependencies
This project will make use of AWS services. As the consumer of those services you are responsible for understanding what costs will be incurred. AWS Billing can be complicated. Most services have a free tier offering but you can easily blow past that if you do not understand how it works. I would suggest that at a minimum, you take a look at two options:
1. Learn how to [manage your AWS billing with Budgets](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/tracking-free-tier-usage.html).
2. [Create a billing alarm](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/monitor_estimated_charges_with_cloudwatch.html) so that you are notified as your charges rack up.

You can further explore the cost structure of each service that is used in this exercise. Most should only incur charges when the API is being consumed, the exception being Route53 which charges a prorated monthly fee for hosting a domain name ($0.50 per domain at the time of writing).

Having said that, all other charges to get this example up and running should be covered by the free tier **if** you account qualifies.

| Service | Usage | Pricing Page |
| ------- | ----- | ----------------- |
| **Cloudformation** | templates defining infrastructure | https://aws.amazon.com/cloudformation/pricing/ |
| **Route53** | DNS hosting | https://aws.amazon.com/route53/pricing/ |
| **Lambda** | serverless on demand compute | https://aws.amazon.com/lambda/pricing/ |
| **IAM** | authentication management | https://aws.amazon.com/iam/ |
| **Certificate Manager** | SSL certificate management | https://aws.amazon.com/certificate-manager/pricing/ |
| **API Gateway** | redundant HTTP gateway management| https://aws.amazon.com/api-gateway/pricing/ |
| **CloudWatch** | logging and metrics management | https://aws.amazon.com/cloudwatch/pricing/ |

## Project Dependencies
Two skeleton Swift projects are used to provide you with sample apps to be installed and run as each Lambda.

| Lambda App | Usage | Repository |
| ------- | ----- | ----------------- |
| **Swift Lambda Authorizer** | allow/deny incoming requests | https://github.com/khinkson/SwiftLambdaAuthorizer |
| **Swift Lambda API** | answer API requests | https://github.com/khinkson/SwiftLambdaAPI |

## Getting Started
Before you get going, know that these instructions were used to setup a live API using the same code and templates shown here. There are more details on how to access this API after the setup instructions. As in the live sample API, the same parameters used will be used throughout the instructions below. When you see the list of parameters below, replace them with your appropriate variable where necessary. The default values in the Cloudformation templates will be the values used in the live demo

| Property Name | Value | Usage |
| ------- | ----- | ----------------- |
| **Cloudformation Role Name** | `SwiftCloudformationRole` | No need to change. |
| **Operations Stack Name** | `operations-stack` | No need to change. If you do, change it everywhere. |
| **WildcardCertificateDomainName** | `openapi.prediket.dev` or `authorizedapi.prediket.dev` | Replace `prediket.dev` with your own domain name. |
| **S3LambdaCodeSourceBucketNamePrefix** | `swiftlambda` | No need to change. Actual bucket name appends the region name eg: `swiftlambda-us-east-2`.|
| **S3LambdaSourceCodeKeyPath** | `SwiftLambdaAPI/lambda.zip` or `SwiftLambdaAAuthorizer/lambda.zip` | No need to change. |

### A. Setup Cloudformation Permissions
1. Log into your AWS console account as a user with administrative permissions and select `IAM` from the main services menu.
2. From the IAM dashboard, select `Roles` from the menu.
3. Click the Create Role button. On the resulting page choose trusted entity type `AWS Service`. From the use cases, choose `CloudFormation`. Hit next, and in the Add Permission table search for and select `AdministratorAccess`.
4. Hit next and name the role `SwiftCloudformationRole`.
5. Click the Create Role button.

It is usually inadvisable to give such broad access to any role or user within IAM . Ideally you should narrow that access permissions to only what is needed by the Cloudformation templates. However, there are a large number of permissions that are required to make these templates work. So for now we will start with a broad permission set.

### B. Setup Domain Name
1. From the AWS Console, select `Route53` from the main services menu.
2. From the Route53 dashboard, select `hosted zones` from the menu.
3. Click the `Create hosted zone`.
4. Enter your domain name as a public hosted zone and complete the creation process.
5. Click on the domain name to see the DNS entries. Look for the DNS entry type `NS` — it will list your name servers.
6. If using a 3rd party domain name provider, go to their dashboard and update the domain name with the new name servers. See your providers documentation if you require assistance.
__NB:__ These changes may take some time to propagate and become available.

### C. Install the Cloudformation Templates
#### The Operations Stack
**Requirements:**
- the Zone ID of your domain name from Route53
- choose your open or authorized subdomain for the wildcard certificate

**Steps**:
1. Select `Cloudformation` from the main services menu.
2. If necessary change to the region where you want to setup the infrastructure. **NB**: Graviton/Arm based lambdas are not available in all regions. If you wish to run ARM only you should [check here](https://aws.amazon.com/lambda/pricing/) first to see if Arm CPUs are available in your regions. Once on that page, scroll down to where the pricing is listed and choose `Arm price`. Select the region to see a list of regions in which Arm is available.
3. Click `Create Stack` (with new Resources if asked)
4. With template ready, choose to upload the template using the project file [0-operations-stack.json](/templates/0-operations-stack.json). Hit next.
5. Enter the stack name `operations-stack`. For the parameters enter the Zone ID of your domain name and the wildcard domain name to use without the astericks prefix. Eg: say `prediket.dev` is you domain, enter `openapi.prediket.dev` or `authorizedapi.prediket.dev` to get a `*.openapi.example.com` or `*.authorizedapi.prediket.dev` domain depending on your choice of open or authorized domain. The wildcard is used because additional domains are created for each region. Click next.
5. Under IAM role choose the previously created `SwiftCloudformationRole`. 
6. Under `Stack creation options` __enable__ `Termination protection`. Click next.
7. Check: `I acknowledge that AWS CloudFormation might create IAM resources with custom names.`
8. Submit. Wait for the this stack to complete creation. The wildcard certificate is created and domain name validation occurs automatically using Route53. This may take more than a few minutes to complete.

#### Lambda Installation
You have already made your choice as to open or authorized API access. You can install an open access API that anyone can use or one that requires authorization. Skip the authorizer step if you want an open API. Remember to use the same domain name chosen for the wildcard certificate.

#### The Swift Lambda Authorizer Stack
**Requirements:**
- Go to the [SwiftLambdaAuthorizer](https://github.com/khinkson/SwiftLambdaAuthorizer) project and follow the instructions on how to build and upload your swift lambda authorizer package.
- the name of your S3 bucket and keyPath where you uploaded the swift lambda authorizer package.

**Steps:**
1. From Cloudformation, click `Create Stack` (with new Resources if asked)
2. With template ready, choose to upload the template using the project file [2.1-authorizer-stack.json](/templates/2.1-authorizer-stack.json). Hit next.
3. Enter the stack name `swift-lambda-authorizer-stack`. For the parameters enter the Zone ID of your domain name. Fill out the rest of the parameters listed. Click Next.
4. Under IAM role choose the previously created `SwiftCloudformationRole`. 
5. Under `Stack creation options` __enable__ `Termination protection`. Click next.
6. Check: `I acknowledge that AWS CloudFormation might create IAM resources with custom names`.
8. Submit. Wait for the this stack to complete creation. This may take more than a few minutes to complete.

#### The Swift API Lambda Stack
**Requirements:**
- the Zone ID of your domain name from Route53.
- the name you entered for the operations stack (`operations-stack` if you followed the instructions above).
- Go to the [SwiftLambdaAPI](https://github.com/khinkson/SwiftLambdaAPI) project and follow the instructions on how to build and upload your swift lambda API package.
- the name of your S3 bucket and keyPath where lambda source code package was uploaded.

1. From Cloudformation, click `Create Stack` (with new Resources if asked). Choose only one (1) of the following:
	- __Open Access Version__: With template ready, choose to upload the template using the project file [2-lambda-open-access-stack.json](/templates/2-lambda-open-access-stack.json). Hit next. 
	__OR__
	- __Authorization Required Version__: With template ready, choose to upload the template using the project file [2.1-lambda-authorization-required-stack.json](/templates/2.1-lambda-authorization-required-stack.json). Hit next.
2. For the authorization required API, enter the stack name `authorized-swift-lambda-api-stack`. For the open access API enter the name `open-swift-lambda-api-stack`. For the parameters enter the Zone ID of your domain name, choose a subdomain prefix for the wildcard API domain. For this example we will choose `main` for the parameter APIDomainPrefix. This will result in `main.openapi.prediket.dev` or `main.authorizedapi.prediket.dev` as our API domain. Fill out the rest of the parameters listed. Click Next.
3. Under IAM role choose the previously created `SwiftCloudformationRole`. 
4. Under `Stack creation options` __enable__ `Termination protection`. Click next.
5. Check off: `I acknowledge that AWS CloudFormation might create IAM resources with custom names`.
6. Submit. Wait for the this stack to complete creation. This may take more than a few minutes to complete.

Once complete you should be able to send a request to your API domain to see if it is working. There are two domains setup for each regional API. https://main.openapi.prediket.dev routes you to the nearest region by latency. In this case there is only one region setup. To access a specific region follow the format — https://main.us-east-2.openapi.predket.dev to access the API infrastructure in us-east-2 (Ohio). Of course replace the region and domain name appropriately to match your setup.

To create another infrastructure stack in a different region, repeat the steps starting from `C. Install the Cloudformation Templates`  in a new region using the same domain name and prefix parameters. You can then test the domain name latency routing or reach each specific region if necessary.

## A Live Example
The APIs, both open and requiring authorization have been setup for anyone who wants to see what the latency is like. These are here to give you an idea of what you can expect from this setup. Both APIs were installed on a fresh AWS account following the exact instructions here. Note: if these APIs get a lot of abuse I will be forced to take them down. To get the curl timing to display, follow the instructions in this [SO answer](https://stackoverflow.com/questions/18215389/how-do-i-measure-request-and-response-times-at-once-using-curl). Both APIs were warmed up first by running a few requests and the authorizer was, and is set to cache it's results for an hour. This test run was not comprehensive. __Your results will vary.__

A successful response looks like this and should contain the region so you know which one you were routed to:
```sh
curl "https://main.openapi.prediket.dev/hello"                                 
{"created":"2023-01-11T18:49:01.386Z","description":"We said hello.","id":"2806E285-ACA9-4E3E-9045-6067334307E8","region":"us-east-2","title":"Hello"}
```

### Open SwiftLambdaAPI

| __Region__ | __Domain__ |
| ------ | ------ |
| **closest by latency** | main.openapi.prediket.dev |
| **us-east-2 (Ohio)**   | main.us-east-2.openapi.prediket.dev |
| **us-west-2 (Oregon)**   | main.us-west-2.openapi.prediket.dev |
| **eu-west-2 (London)**   | main.eu-west-2.openapi.prediket.dev |
| **eu-central-1 (Frankfurt)**   | main.eu-central-1.openapi.prediket.dev |
| **ap-southeast-1 (Singapore)**   | main.ap-southeast-1.openapi.prediket.dev |
| **sa-east-1 (São Paulo)**   | main.sa-east-1.openapi.prediket.dev |
| **ap-northeast-1 (Tokyo)**   | main.ap-northeast-1.openapi.prediket.dev |

From an EC2 instance in us-east-2 (Ohio) to the Open SwiftLambdaAPI in us-east-2 (Ohio) — approximately 37ms to get a response:

```sh
curl -w "@curl-format.txt" \
-o /dev/null \
-s "https://main.openapi.prediket.dev/hello"

          DNS Lookup:  0.001086s
 Remote Host Connect:  0.001606s
       SSL Handshake:  0.010577s
   Pretransfer Setup:  0.010671s
       Any Redirects:  0.000000s
 First Byte Transfer:  0.037326s
                     ----------
               Total:  0.037417s
```

### Authorized SwiftLambdaAPI

| __Region__ | __Domain__ |
| ------ | ------ |
| **closest by latency** | main.authorizedapi.prediket.dev |
| **us-east-1 (N. Virginia)**   | main.us-east-1.authorizedapi.prediket.dev |
| **us-west-1 (California)**   | main.us-west-1.authorizedapi.prediket.dev |
| **eu-west-1 (Ireland)**   | main.eu-west-1.authorizedapi.prediket.dev |
| **ap-south-1 (Mumbai)**   | main.ap-south-1.authorizedapi.prediket.dev |
| **eu-west-3 (Paris)**   | main.eu-west-3.authorizedapi.prediket.dev |
| **ap-southeast-2 (Sydney)**   | main.ap-southeast-2.authorizedapi.prediket.dev |
| **ap-northeast-2 (Seoul)**   | main.ap-northeast-2.authorizedapi.prediket.dev |

From an EC2 instance in us-east-1 (N. Virginia) to the Authorization Required SwiftLambdaAPI in us-east-1 (N. Virginia) — approximately 40ms to get a response:

```sh
curl -w "@curl-format.txt" \
-o /dev/null \
-s  "https://main.authorizedapi.prediket.dev/hello" \
-H 'Content-Type: application/json' \
-H 'Authorization: SKOcSFZd8pKoZ24WXYK2dSEc5Nf2eao9QOOvpPYM'

          DNS Lookup:  0.001171s
 Remote Host Connect:  0.002046s
       SSL Handshake:  0.011209s
   Pretransfer Setup:  0.011324s
       Any Redirects:  0.000000s
 First Byte Transfer:  0.040475s
                     ----------
               Total:  0.040564s
```

## Next Steps
As mentioned above this project is part of the [Swift in the Cloud Series](https://www.flue.cloud/swift-server-cloud/). The next step in the series deals with [DynamoDB](https://aws.amazon.com/dynamodb/) — a multi-regional database (key value NOSQL database) that you can use in conjunction with the multi-regional lambda API. 

## Further Improvements
Time will bring changes to this project. To really make this setup robust, it can be improved by adding:
1. Allow templates to be reused to build multiple APIs in the same account/region
2. Automated continuous integration/deployment (CI/CD) (AWS Pipeline)
3. A firewall for filtering malicious traffic (AWS Web Application Firewall)
4. Automated failover to reroute traffic to healthy regions (Route53 DNS Health Checks)
5. Detailed metrics collection, aggregation and analysis (AWS CloudWatch)
6. Possibly convert the templates to a StackSet (AWS Cloudformation)
7. Overall automation via the AWS CLI

## License
__AWS Swift Multi-Region API__ is released under the [Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0). See LICENSE for details.
