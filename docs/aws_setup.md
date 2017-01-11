# A Brief Interlude Regarding AWS Setup

Presumably, since you've read this far, you're interested in more than a simple Django powered API serving static images.  You probably want something that is more interactive with users and have have advanced capabilities.  So this will require interacting with additional AWS services like RDS or DynamoDB for database services, SNS or SQS for task processing, and many more...

Well, there is an important aspect about interacting with additional AWS services.  With pure AWS lambda, we have a fairly low attack surface, but when we introduce additional interactions over the network, we must consider information security risks.

## A simple, but naive approach

A simple approach would be to create an AWS RDS instance that is open to the Internet and have your lambda Django project login directly.  While this may work, this approach is fraught with peril.  Numerous vulnerabilities could exist and credentials could be brute forces.  Even without access to your RDS, it is trivial to launch a denial-of-service attack to ensure your Django project has no database services.

Basically, do not do this.

## That's why Amazon has VPC

Introduce Amazon Virtual Private Cloud (VPC).  This provides a way of segmenting Internet traffic from your AWS services.  This feature is incredibly valuable to securing our project.  So that bad guys have the smallest attack surface possible.  It's easy to setup a VPC - there are many webpages that will assist you.  But basically you pick an IP range and let it fly.

Once a VPC is established, you can then subdivide the VPC into subnets.  Which are little non-overlapping IP chunks of the overall VPC.  Like dividing a pie into slices.  

It is important to realize that you have almost entire control of your IP space when you define your VPC and associated subnets.  You are creating a Software Defined Network (SDN) and you have a lot of leeway.  With this freedom comes choices and important considerations on how you design your network.

For a full list of options and other valuable information about VPCs, see this link: [https://aws.amazon.com/answers/networking/aws-single-vpc-design/](https://aws.amazon.com/answers/networking/aws-single-vpc-design/)

The options are summarized here:

* Internet-Accessible VPC

    One could serious consider this configuration.  If you put your lambda functions, your RDS, and additional SNS/SQS services in a single subnet in an Internet Accessible VPC it could work.  In theory you could configure your security groups to ensure only lambda functions can hit your RDS.  
    
    One advantage of this setup is that you can setup your local machine to connect to your RDS without a [bastion host](http://serverfault.com/a/648116/60094).  Just restrict access based on IP.  
    
    **This scenario could be very viable based on your needs**

* Public and Privately Routed VPC

    Arguably the most secure of all web application setups.  You have two subnets: one public and one private.  Your RDS and lambda functions reside in the private subnet, far away from bad guys, but also far away from your local machine.  In order to access the database from your system you will need a [bastion host](http://serverfault.com/a/648116/60094) in the public subnet.  

	Another advantage of this setup is that if you ever want to add additional EC2-based services that need to interact with the Internet you can do this very easily without compromising security.  

	**The upshot is this configuration is the most secure and most flexible for your growth needs with minimal additional complexity.**

* On-Premises and Internet-Accessible VPC

	Same as the last configuration, but if you have an internal corporate network to connect, you can easily establish a connection to the private subnet without compromising security.

	**A good solution if you need to connect an internal corporate network**

* Internal-Only VPC

	Obviously a very special case of creating a Django app for internal use only with no desire to have it accessible by the Internet.
	
	**Useful for special cases**
	
