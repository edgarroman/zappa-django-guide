# Adventures in Networking

Presumably, since you've read this far, you're interested in more than a simple Django powered API serving static images.  You probably want something that is more interactive with users and have have advanced capabilities.  So this will require interacting with additional AWS services like RDS or DynamoDB for database services, SNS or SQS for task processing, and many more...

Well, there is an important aspect about interacting with additional AWS services.  With pure AWS lambda, we have a fairly low attack surface, but when we introduce additional interactions over the network, we must consider information security risks and our network architecture.

!!! Note
    This section assumes some familarity with the basic concepts within AWS VPC.  If you aren't comfortable with these concepts,
    then head over to a [Primer on AWS VPC Networking](aws_network_primer.md)

## A simple, but naive approach

A simple approach would be to create an AWS RDS instance that is open to the Internet and have your lambda Django project login directly.  While this may work, this approach is fraught with peril.  Numerous vulnerabilities could exist and credentials could be discovered by brute force.  Even without gaining access to your RDS, it is trivial to launch a denial-of-service attack to ensure your Django project has no database services.

Basically, do not do this.

## That's why Amazon has VPC

Introducing AWS Virtual Private Cloud (VPC).  This service provides a way of segmenting Internet traffic from your other AWS services.  This feature is incredibly valuable to securing our project, so that bad guys have the smallest attack surface possible.  

Once a VPC is established, you can then subdivide the VPC into subnets.  Which are little non-overlapping IP chunks of the overall VPC.  Like dividing a pie into slices.

It is important to realize that you have almost entire control of your IP space when you define your VPC and associated subnets.  You are creating a Software Defined Network (SDN) and you have a lot of leeway.  With this freedom comes choices and important considerations on how you design your network.

## The Essential VPC and Subnet Configuration

Here is the absolute minimum you must have to have a working Django site within a VPC:

* A VPC
* A subnet within the VPC
* A security group
* Your zappa project configured to use the subnet and security group

There are no need for routes, NAT gateways, Internet gateways, or even allowing inbound rules on the security group.  You will probably need many of these services eventually for a robust and full-featured application, but for illustration purposes now, we will keep it simple.

You may be asking yourself how will traffic from the Internet reach the lambda function. Well essentially the magic is in the API Gateway that zappa creates on deployment.  From the [API Gateway FAQ](https://aws.amazon.com/api-gateway/faqs/):

> Amazon API Gateway endpoints are always public to the Internet. Proxy requests to backend operations also need to be publicly accessible on the Internet. However, you can generate a client-side SSL certificate in Amazon API Gateway to verify that requests to your backend systems were sent by API Gateway using the public key of the certificate.

So API Gateways act as an 'always-on' Internet facing service that sends the HTTP requests to the Zappa Lambda functions by creating 
Lambda events.  These events can reach your zappa application even if deployed in a top-secret lockbox.  Note that in the above VPC setup the Lambda function is completely isolated from direct access to the Internet.  Thus the Lambda functions can't make outbound connections to anything: other servers, databases, S3, etc.

Let's face it, there's no real point to going through the effort of doing this if the Lambda functions are really this isolated.  
*But* the reason this information is important is because in order to leverage other AWS resources, 
the VPC lays the foundation to do this securely.

## Extending the VPC

To re-iterate: we don't need a VPC network to make the Zappa Lambda-powered Django site visible on the Internet.  
We need to extend the VPC so that we can add more AWS services like:

* Adding an RDS database that is only accessible from our Lambda functions
* Adding an S3 bucket 
* Allowing Lambda functions to communicate to traditional EC2 instances for long running processes
* Allowing the Lambda functions to access the Internet to hit an external API
* Ability to interact with SQS and/or SNS

The important thing is that we can enable all these scenarios in a secure manner.  There are too many scenarios to go into detail here, so we will provide some guidelines.

Next we learn about the common patterns of VPC usage and try to identify usage options.

One warning before we go on with VPC subnet sizing and Lambda:  When Lambda functions fire, they must be assigned an IP address temporarily.  If you have a lot of Lambda functions firing, then you must have a lot of IP addresses available in your subnet.  You can learn more [here](http://docs.aws.amazon.com/lambda/latest/dg/vpc.html#vpc-setup-guidelines).

## VPC Patterns

While the actual VPC itself is very straightforward, the combination of subnets within the VPC can take many forms.  Additionally, you must consider how the subnets will communicate among themselves as well as how  they communicate with outside networks. For a more comprehensive list of options and other valuable information about VPCs, see this link: [https://aws.amazon.com/answers/networking/aws-single-vpc-design/](https://aws.amazon.com/answers/networking/aws-single-vpc-design/)

The options are summarized here:

### VPC with a single Internet-Accessible subnet

This pattern places your lambda functions, your RDS, and additional SNS/SQS services in a single subnet that is Internet accessible in your VPC.  In theory you could configure your security groups to ensure only lambda functions can hit your RDS.

One advantage of this setup is that you can setup your local machine to connect to your RDS without a [bastion host](http://serverfault.com/a/648116/60094).  Just restrict access based on IP.  

Important note - you will want to ensure careful inbound IP restrictions.  While it's great that you can connect to RDS with your SQL desktop client, you should setup a [bastion host]((http://serverfault.com/a/648116/60094)).

Also note that your zappa deployment 
[will not have outbound access to the Internet](aws_network_primer.md#general-behavior-and-internet-access).
In order to do this you will have use a Public/Private setup.

**This scenario is good for straightforward setups with a little work, but has significant limitations**

### VPC with a Public subnet and Private subnet

Arguably the most flexible and future-proof of all web application setups.  You have two subnets: one public and one private.  Your RDS and lambda functions reside in the private subnet, far away from bad guys, but also far away from your local machine.  In order to access the database from your system you will need a [bastion host](http://serverfault.com/a/648116/60094) in the public subnet.  

Another advantage of this setup is that if you ever want to add additional EC2-based services that need to interact with the Internet you can do this very easily without compromising security.  

Generally this setup will require networking knowledge to setup the Internet gateway, bastion host, and NAT Gateway.

**The upshot is this configuration is the most secure and most flexible for your growth but will be complex from a network standpoint**

### On-Premises and Internet-Accessible VPC

Same as the last configuration, but if you have an internal corporate network to connect, you can easily establish a connection to the private subnet without compromising security.

If you thought the last setup was complex, you better know what you are doing from a network standpoint.

**A good solution if you need to connect an internal corporate network**

### VPC with an Internal-Only subnet

Obviously a very special case of creating a Django app for internal use only with no desire to have it accessible by the Internet.

This is actually most secure.  Since you can access any of the resources from your desktop on the internal network.  Not bad for the paranoid or security conscious devops team.

**Useful for the simple environment if you have an existing secure network**
	
## Subdividing the VPC

Once you get a VPC selected you must create subnets within the VPC.  When defining a subnet, you just have to pick a non-overlapping segment of the ip range.  So if you have VPC that spans IP address 10.5.0.1 to 10.5.0.254, then you pick contiguous segments within this range.

See more details in [Primer on AWS VPC Networking](aws_network_primer.md)

## Examples for Walkthroughs

For the purposes of walkthroughs, we will leverage a simple VPC with a single subnet.  A single subnet will generally be enough to guide readers through the scenarios.

We have a VPC:

* id: vpc-9a9a1dfc
* cidr: 10.6.0.0/16

With subnet:

* id: subnet-f3446aba
* cidr: 10.6.1.0/24

And security group:

* id: sg-13a5736f
* inbound rules: none
* outbound rules: all traffic

> TODO: Show example zappa configuration here


## Note on Redundancy

While these examples are all using a single subnet for clarity, in production you will want to create multiple subnets within the VPC all with different availability zones.  This ensures if there is a failure within a single availability zone, there are alternate paths.  

The general approach is to associate the Lambda functions with multiple subnets and the AWS resources with the same multiple subnets (e.g. RDS).