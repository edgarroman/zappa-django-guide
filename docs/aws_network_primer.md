# A Brief Primer on AWS VPC Networking (and how Lambda functions relate)

Confused on how to create your network environment in AWS?  Don't worry - you're in good company.  Configuring and using AWS VPC networking 
is powerful, but complex.  

This document attempts to create a [mental model](https://en.wikipedia.org/wiki/Mental_model) to help readers understand the concepts and thus 
allow more reasonable decisions to be made.  I find it helpful to draw analogies to traditional network setup within a company to assist building
the mental model.  

If you're already familiar with AWS VPC, you can skip to [how AWS Lambda interacts with VPC](#how-lambda-works-with-vpc)

This document takes the general approach:

* First, discuss the concepts and talk about how they relate
* Link to good tutorials on how to create VPCs once the reader has a good understanding

We won't go into exacting detail on how to do each step since there are many good tutorials already on the Interwebs.

## First there is a VPC

Back in the bad old days, networks were created by plugging a bunch of network cables into network hardware and were statically configured.  
In AWS, the amazing thing is that you define the network dynamically by providing parameters to Amazon as a form of [Software Defined Networking](https://en.wikipedia.org/wiki/Software-defined_networking).  So Amazon has hardware in their datacenters but present to users a very dynamic environment is nearly indistinguishable from an old-school network.

The first thing most users do is define the overall boundaries of their network using a Virtual Private Cloud or VPC.  This is roughly akin to
drawing a circle around your infrastructure.  The old-school equivalent would be to define your company's internal network space.  This network
space would allow your various departments to have computers that belong to this space.  Old-school companies might have several floors and departments
so it would have to be big enough to house all these computers.  Additionally, you don't want external hackers have unfettered access to your 
internal databases and accounting systems so most companies pick [Private Network Space](https://en.wikipedia.org/wiki/Private_network) to
isolate the big bad Internet from the company network.

You do this in AWS VPC by defining the IP network space of the total possible IP addresses that *could* live in the VPC.  Now you may not have 
lots and lots of computers you'll need to put into your VPC, but fortunately you are not charged by Amazon on how many IP addresses
you have reserved, so you can err on the side of having a bit of room.

The private network ranges available are defined by an Internet 'standard' called [RFC 1918](http://www.faqs.org/rfcs/rfc1918.html).  The private network spaces for IPv4 are:

* `10.0.0.0 - 10.255.255.255` (10/8 prefix)
* `172.16.0.0 - 172.31.255.255` (172.16/12 prefix)
* `192.168.0.0 - 192.168.255.255` (192.168/16 prefix)

AWS restricts creation of VPCs with a /16 CIDR mask which limits you to a VPC with 65,536 IP addresses (which is pretty big).  This is probably a little excessive for your first VPC, so why not start with something like 10.0.0.0/20 which gives you about 4 thousand IP addresses?  Later we'll keep dividing this network space so this is a good start.

### Now create your subnets

Ok, so now we've got a range of IP addresses that we can use and a potential route to the Internet.  Using our old-school network analogy,
what a typical network engineer would do next is subdivide the network into chunks.  And the more technical term of a chunk of the 
network is a [subnet](https://en.wikipedia.org/wiki/Subnetwork).  In physical world, some possible reasons do this are:

* If a building has multiple floors, maybe one subnet per floor
* Maybe divide a subnet for each business department (e.g. HR subnet, IT subnet, Software Development subnet)
* Subnet based on functionality: one subnet for the phone system, one subnet for desktop computers, one subnet for web servers

In the VPC world, the only real reasons needed to divide up the VPC is for functionality and security.  A very common example
would be if you want web application servers in a public subnet and database servers in a private subnet.  In this configuration,
users on the Internet can point their browsers at your website but cannot directly access your database servers.  And you can
create a special connection from your public subnet to your private subnet so that only the web application servers can connect
to the database servers.  In this way, you lessen the chances that bad actors can attack your database by restricting direct access.

#### Notes on subnet calculations

There are a lot of complex rules for how big and location of the subnets  within the VPC, but there are two important constraints:

* The size of the subnets must be a power of 2 - which approximately results in the number of IP addresses assigned to the subnet 
  (e.g. 2, 4, 8, 16, 32, 64, 128, 256, etc IP addresses available)
* The subnets must be contiguous - which means the available IP addresses in the chunk must be sequential

By far, the easiest way to visualize this system is to use a tool that handles all the complications for you.  I highly recommend
the [Spiceworks Subnet Calculator](https://community.spiceworks.com/tools/subnet-calc/)

#### But there's more

Just a few more notes about subnets created within your VPC.  First, each subnet exists in one availability zone within one AWS Region.
For your experimentation, this should not be a big deal.  But once you have a production system that is designed to be highly-available,
then you will probably have to eventually create multiple subnets that are doing the same function (e.g. hosting a database or application
servers) to guard against the case when one availability zone is having troubles, your infrastructure still is up. See the 
AWS documentation for more info on [Regions and Availability Zones](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html).

Each subnet has a built-in [Access Control List (ACL)](https://en.wikipedia.org/wiki/Access_control_list#Networking_ACLs).  This will let
you control inbound and outbound network traffic at the TCP/IP level.  Thus you can create rules that allow or restrict network traffic
that applies to the entire subnet.  This is an important distinction from a more robust firewall or security device: regardless of how
many servers you may have in your subnet, they all share these rules.  So if a particular external IP address is permitted in the ACL,
then all servers could generate traffic to that external IP address.  If more fine-grained control per resource
is necessary, see [VPC Security Groups](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_SecurityGroups.html).

Lastly, there is a [limit of 200 of subnets per VPC](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Appendix_Limits.html#vpc-limits-vpcs-subnets)
 imposed by AWS at the time of this writing.

### Hooking things up: route tables

Usually you'll want various subnets to communicate -- and by default, AWS allows all traffic to flow from any subnet to any another.  So that's
convenient but usually we want a better security posture.

AWS VPC uses route tables to figure out how network traffic should be shuffled around.  A route table is a configurable object within the AWS
console.  When you create a new VPC, the AWS system will automatically create a route table for you and assign it to your VPC.  A VPC is 
always assigned to a route table; and that assigned route table is the 'main' route table.  But there is no star, no icon, or anything special
in the AWS console that this is a 'main' route table -- merely the fact that when you click on the VPC, only one route table will be listed there.
And you cannot change a VPC 'main' route table.

So this 'main' route table has some special properties.  Turns out that each subnet is also assigned a route table.  And if you don't explicitly
assign a route table on creation of the subnet, it gloms onto whatever the current 'main' route table is. So the 'main' route table is
used as a default for all subnets that aren't assigned a specific route table.  Thus all the subnets will share a single route table by 
default unless you take action.  By taking action, I mean you can assign a different route table to a subnet.

#### Why do we need multiple route tables?

The question you may be asking yourself is why do we even need more than one route table?  The most common usage is to increase security by 
specializing the subnets.  Since we know all the subnets are really just networks, there is no functional difference until we modify how
network traffic flows between the subnets.  Back to the example of web app servers and databases.  We can put all the web app server 
instances in Subnet A and the database server instances in Subnet B.  But then by restricting the routes so that Subnet B can only talk
to Subnet A, we can consider Subnet B as 'private'.  Again, there is no sticker, no emjoi, nor icon in the AWS console that designates 
Subnet B as 'private' except for the fact that we've associated a restricted route table.  Conversely, we can allow Internet traffic to go
out of Subnet A by assigning a different route table; thereby creating a 'public' subnet in name only.

### Talking to the Internet

Usually, most applications will need to communicate with the Internet; either taking incoming network connections 
or retrieving information. In AWS you connect your VPC to the Internet using 
[Internet Gateways](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Internet_Gateway.html). 
This is a special resource that is assigned to a VPC that allows network traffic to come in and out of your VPC.
Internet Gateways are free to AWS users and can be created easily by using the VPC creation wizard or after the fact.
An Internet Gateway can only be attached to a single VPC at a time and a VPC can only have up to one IG attached at a time.  
But VPCs do not have to have Internet Gateways attached.

But creating an Internet Gateway and then associating it with a VPC is only the preparation work.  In order to actually
enable traffic into and out of a subnet, you must have a route associated with the subnet that connects the Internet Gateway.
So for the subnet you wish to designate as 'public' you must add a route:

* Destination: `0.0.0.0/0` - This is a special indicator to the route table that works as a 'catch-all' for any network
traffic destination not recognized
* Target: `igw-name-of-your-ig` - Use the AWS console identifer of the IG associated with the current VPC

And presto!  You can now consider any subnets associated with this route table as 'public' subnets.  Any EC2 instance
that is assigned an Elastic IP can now send and receive traffic from the Internet.  Any EC2 instances without an
Elastic IP will not be permitted to send and receive traffic.

#### Other AWS Services

Note that many of the AWS services are not associated with any VPC and thus can be considered accessible via Internet-only.
For example, if your application living in the VPN needs to connect to the AWS Simple Queue Service (SQS), you will have to 
enable Internet access via an Internet Gateway as described above.
Any AWS service that does not have a [VPC endpoint](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/vpc-endpoints.html) capability is considered Internet only.
At the time of this writing, only the following services have [VPC endpoints](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/vpc-endpoints.html)
 (and thus would not need an Internet Gateway):

* S3
* DynamoDB
* [ElasticSearch](http://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/es-vpc.html)

In addition, some services like AWS RDS and ElastiCache can be assigned to VPC subnets 
and thus are natively accessible within a VPC.  These services effectively provide fully 
managed EC2 instances that provide the services and thus can be assigned to one or more VPC subnets.

### Connectivity from private subnets

Often, resources in 'private' subnets will have to communicate to the Internet.  Usually, this network
traffic is initiated from within the subnet and only return traffic is allowed.  If outside network traffic were permitted
to initiate communication to 'private' subnet servers, it would expose a potential security risk and thus is not allowed.

The mechanism to allow servers and resources within a 'private' subnet to have outbound communication is to 
have the network traffic sent to a special resource called a NAT device.  NAT stands for [Network Address Translation](https://en.wikipedia.org/wiki/Network_address_translation) 
and is generally used to hide a number of private servers or resources from any outside network.  This NAT device
must live in a 'public' subnet with access to the Internet since it acts as a middle-man for the network traffic: 
the private resources network traffics gets sent from the 'private' subnet to the 'public' subnet and the NAT
device passes along the data as if the NAT device was initiating the connection.

Enabling a server or resource in a 'private' subnet to communicate to the Internet consists of:

1. Create the NAT Device (more on that below) in a 'public' subnet
1. Edit the route table assigned to the 'private' subnet to have a new route:
   * Destination: `0.0.0.0/0` - This is a special indicator to the route table that works as a 'catch-all' for any network
traffic destination not recognized
   * Target: `nat-name-of-your-nat-device` - Use the AWS console identifer of the NAT device in the 'public' subnet

Note this will enable any server or resource in the 'private' subnet to use the NAT device for outbound communication.

#### Types of NAT Devices

NAT Devices are not free and the cost to the AWS account depends on the type of device.  To assist
the decision process, AWS has provided a guide on common types:

* [NAT Gateways](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/vpc-nat-gateway.html) - these
  are AWS-managed instances that are easy to spin-up and have minimal configuration.  This is the
  easiest option for getting started.
* [NAT Instances](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_NAT_Instance.html) - these are based on an existing AWS AMI that creates an EC2 instance that you can further customize.

AWS even provides a [comparison chart](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/vpc-nat-comparison.html)
to help you decide.

#### Other AWS Services

Recall that many AWS services are not accessible directly within a VPC.  So if your applications
depend on other AWS services such as SNS, SQS, SES, and so forth, a NAT device will be required
at additional cost.

### Advanced Topics

There are many, many advanced configurations available with [VPC](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Introduction.html):

* [DNS](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/vpc-dns.html)
* [VPN Connections](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/vpn-connections.html)
* [DHCP](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_DHCP_Options.html)
* [VPC Peering](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/vpc-peering.html)

But none of these are required to get a basic VPC setup rolling, so they are left as an exercise for
the reader.

### Next Steps

I hope you have a solid mental model of the basic building block of AWS VPC and thus can understand the relationships
between the myriad of concepts presented in the AWS documentation.  With this mental model, I recommend reading 
carefully through the well-documented [VPC Scenarios](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Scenarios.html)
 on the AWS documentation site.

## How Lambda Works with VPC

Now that we've covered the basics with typical resources within a VPC, it's worth discussing
how AWS Lambda interacts with a VPC.  While it seems to be fairly straightforward, there is a surprise feature
with Lambda that bypasses the entire network model.

### Multiple Subnets allowed

When creating a Lambda function, you can assign zero, one, or multiple subnets.  If no subnets are specified, then the
Lambda function is considered external to all VPCs in the account.

The reason for multiple subnets would be mostly for robustness.  When a Lambda container is started, it must be assigned an
IP address.  It will select the IP address from one of the assigned subnets.  The exact method for selection is not documented.
The Lambda container will run, then release the IP address.  But if you have many Lambda containers running, there is a 
risk of using up all the available IP addresses.  In addition, since subnets are assigned a single availability zone (AZ),
by associating subnets in a few different AZ, you can increase the chances that your Lambda function will continue to
run even if one AZ is having troubles.

The downside to assigning multiple subnets is you must ensure the routing rules are identical for both subnets otherwise
you could create a situation where Lambda containers may behave differently based on the dynamically assigned subnet.  This
would be very difficult to troubleshoot.

### General Behavior and Internet Access

Lambda functions running in a VPC mostly have all the same internal network access like any other resource in the subnet.
However, Lambda functions are dynamically assigned an [Elastic Network Interface (ENI)](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_ElasticNetworkInterfaces.html) which acts just like
a typical IP address within the VPC.  But ENIs are not publicly routable, so the Lambda function will not be able to access
the Internet directly.  Even if the Lambda function is assigned to a 'public' subnet with access to an Internet Gateway, it 
will not be able to leverage the Internet Gateway.  As per [AWS Documentation](http://docs.aws.amazon.com/lambda/latest/dg/vpc.html#vpc-internet) if you want Lambda functions to access the Internet,
you must assign those Lambda functions to a 'private' subnet and leverage a NAT device as described above.  This includes
the scenario when the Lambda functions needs to interact with other AWS services that are considered only available
via the Internet.

Aside from that, Lambda functions that do not require Internet access can happily operate in any of your VPC subnets.

### Bypassing your VPC setup

It is important to remember that Lambda functions are a separate and distinct AWS service from VPC.  VPC merely defines
the IP network space in which a Lambda function will operate.  The heart of the Lambda service could be considered
the ability to be triggered from an event and optionally respond to the event.  These events come from various sources
and services within the AWS fabric.  It can be described as an 'event fabric' consisting of large number 
of Lambda functions listening for events, processing them, and responding.  
In fact, this method of communication is completely separate and distinct from network traffic.  It has its own
security model and retry mechanism -- very much like a point-to-point network.  

The implications of leveraging this 'event network' is that it can be used to effectively 'tunnel' across separate
and distinct VPC and subnets.  It takes a little bit of work but consider:

* Lambda X deployed outside of any VPC in your account
* Lambda Y deployed in a 'private' subnet within the VPC
* RDS instance in 'private' subnet within the VPC
* API Gateway deployed in your account (also outside any VPC)

If properly configured the following sequence could happen:

1. API Gateway creates an event for Lambda X
1. Lambda X gets the event, then creates an event for Lambda Y
1. Lambda Y gets the event, then makes a query to the RDS database via traditional network mechanisms
1. Lambda Y returns the information to Lambda X
1. Lambda X returns the information to API Gateway

Now this is clearly a contrived example as API Gateway could actually invoke Lambda Y directly.  But it is meant
to illustrate that with some limitations, there are ways to get around the VPC constructs.
A more useful scenario would be to have Lambda Y invoke the Lambda X function to gain access to 'Internet only'
AWS services such as SQS without the need for a NAT Device.  There are many complications with this strategy:

* You now have to juggle two different authorization schemes - Lambda and traditional access methods
* You have doubled the number of Lambda function invocations which could lead to additional charges
* You have a execution limit on both the number simultaneous of Lambda functions and length of time allowed
  per invocation
* You have to create and manage the event chain manually

In my opinion, it is important to know this functionality exists, but in almost all cases, it would be more
cost effective to leverage one of the NAT Device strategies outlined above.

At the time of this writing, the following events can 
[generate events for Lambda functions](http://docs.aws.amazon.com/lambda/latest/dg/invoking-lambda-function.html):

* Amazon S3
* Amazon DynamoDB
* Amazon Kinesis Streams
* Amazon Simple Notification Service
* Amazon Simple Email Service
* Amazon Cognito
* AWS CloudFormation
* Amazon CloudWatch Logs
* Amazon CloudWatch Events
* AWS CodeCommit
* Scheduled Events (powered by Amazon CloudWatch Events)
* AWS Config
* Amazon Alexa
* Amazon Lex
* Amazon API Gateway

## Final considerations

It takes some planning when planning to deploy a Zappa environment and to determine if VPC is required or necessary.
In the [next section](aws_network.md), we discuss some concrete options for the walkthrough.
