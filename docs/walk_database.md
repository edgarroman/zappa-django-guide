# Using a Database

This walkthough documents the steps necessary to connect your application to a hosted database.

## Prerequisites

This walkthough requires the (Core Django Setup)[walk_core.md] to be completed.  Also, it is important 
to have your network setup properly so check out (Adventures in Networking)[aws_network.md].  

We will assume you have chosen the VPC pattern: "VPC with a Public subnet and Private subnet"
But basically you will need the private subnet or subnets which can access the database.

## Options for Databases

### Use AWS RDS

This is probably the easiest to get up and running.  AWS takes care of the messy details of managing the host and provides database-as-a-service (if that's a real thing).  In addition, AWS RDS supports mySQL and PostGreSQL, both which are highly compatible with Django.  

### Host Your Own

Of course you can be running any type of database on an EC2 instance of your choosing.  Usually an EC2 instance will be associated with one subnet, but it is possible to have multiple IP addresses in different subnets for redundancy.

### Use another DB Service

There as some other database services such as DynamoDB.  Depending on the capabilities of the service, you may or may need the subnet information.  

## Let's do this thing

We'll just focus on the RDS case for this walkthough.  In fact we'll go through the walkthough using PostGreSQL.

create db subnet in two availablilty zones

Then create db - new security group




http://marcelog.github.io/articles/aws_lambda_internet_vpc.html
https://www.isc.upenn.edu/accessing-mysql-databases-aws-python-lambda-function