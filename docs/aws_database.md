# Creating an RDS Database

There are a number of RDS engines available - https://aws.amazon.com/rds/getting-started/. Another choice is to opt out for DynamoDB - https://aws.amazon.com/rds/getting-started/ 

## Adding second private subnet to VPC

RDS requires additional subnet to create an instance. Follow step 3 **Create Additional Subnets**
 at https://docs.aws.amazon.com/AmazonECS/latest/developerguide/create-public-private-vpc.html to add one to your existing VPC. Choose an **Availability Zone** different from the first one and **IPv4 CIDR block** 10.0.2.0/24.

## Creating an RDS instance

Follow this guide https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_GettingStarted.CreatingConnecting.PostgreSQL.html keeping in mind parameters we created before.
