# AWS capstone project - A 3 tier database service

## Scenario:
You have a web application that accepts requests from the internet. Clients can send requests to query for data. When a request comes in, the web application queries a MySQL database and returns the data to the client.

Design a three-tier architecture that follows AWS best practices by using services such as 
- Amazon Virtual Private Cloud (Amazon VPC), 
- Amazon Elastic Compute Cloud (Amazon EC2),
- Amazon Relational Database Service (Amazon RDS) with high availability,
- and Elastic Load Balancing (ELB).

## Initial analysis:
From the requirements it can be seen that there is a need for a webserver system, a database (RDS) system and load balancing with ELB. There is no requirement given for direct remote access to the systems. The basic layout for the environment would then be:

internet <=> load balancer <=> webservers <=> databases

There is no stated security level for this service so I will attempt to make it as secure as possible. The database is to have no direct exposure to the internet and the webserver systems should have adequate network protection.

## Solution:
There are to be three tiers to the environment, an internet/web tier, an application tier and a database tier. As the service is to be internet-accessible the web tier will be hosted in a public subnet.

To reduce security threats, the application tier and the database tier will hosted in private subnets.

The layout is:
web tier - public subnet - elastic loadbalancer
application tier - private subnet - EC2 instances
database tier - private subnet - RDS(MySql), with a R/W primary mirrored to a R/O secondary

The proposed structure is shown below:
[3 tier AWS structure](3tier.jpg)

### Network layout:
A new VPC is to be created with an address space of 10.0.0.0/16. This provides ample address space to provide distinct subnets as well as the ability for easy future expansion of the service if required.

For this inital setup, the VPC will span two availability zones (AZ-1, AZ-2). There will be five defined subnets, one public and four private.

The VPC is notionally subdivided into a 'networking layer', an 'application layer' and a 'database layer'

### Network layer
The 'networking' layer is public-accessible through the internet gateway, tagged 'igw'.

An application load balancer (ALB) with integrated web application firewall is hosted in this layer. The ALB will pass traffic between the internet gateway (igw) and a target group (app-trg-grp). The ALB provides loadbalancing as well as protection against DDOS attacks. The integrated web application firewall protects against known layer-7 exploits.

This layer uses the Route53 DNS service to track the IP addresses of the ALB instances across the availability zones. It will also publish the public IP address for accessing this setup. 

This layer has a route table (RT-1) that contains a default route to the internet via the internet gateway (igw), and to the local address space of 10.0.0./16.

The ALB has an attached security group, net-sec-grp, which contains rules as to which network services are available. This security group should allow incoming connections for HTTP (port 80), HTTPs (port 443) and MySQL (port 3306).

### Application layer
The next layer is the 'application layer'. It is made up of two private subnets (10.0.4.0/24, 10.0.5.0/24) which are each hosted in separate AZs (AZ-1, AZ-2). Each subnet hosts an EC2 instance that provides a webserver. The EC2 instances are part of the target group (app-trg-grp).

This layer hosts the EC2 servers that provide the public service. All EC2 servers are members of a security group (app-sec-grp). This security group has rules to allow only connections for ports 80, 443 and 2206 from the network layer security group (net-sec-grp) and deny all other connections. This reduces the risk of unauthorised, unplanned access to the web and data servies provided.

The routing table for this layer, RT-2, allows it to communicate with entities within the VPC and the wider internet. It should hold the following rules:
~~~ 
<span style="background-color:DarkGray">
10.0/16 local
0.0.0.0/0 igw</span>
~~~ 
## Database layer
The last layer is the 'database layer'. It is made up of two private subnets (10.0.8.0/24, 10.0.9.0/24) which reside in the seperate availability zones. This layer holds an RDS instance with a 'primary' server hosted in one availability zone and a 'backup' server hosted in the other. The database instances are part of a security group (db-sec-grp) that allows access from the application security group (app-sec-grp) but denies all other access. The database servers can connect to each other within the security group.

The 'primary' database instance is run in read/write mode so that all transactions can be recorded and queried. The 'backup' server only takes write operations from the primary. This should reduce the risk of any externally triggered corruption of the primary database being propagated to the backup database. This also makes the backup server a "hot backup", able to be rapidly promoted to replace the primary database if needed.

The backup server is also available for read operation requests from the application layer servers. This provides extra capacity to the application servers by handling read-only database queries.

To restrict network traffic directly to/from the internet route table RT-3 should be configured to only interact with the local network. It hold have only one entry, which is:
~~~ 
<span style="background-color:DarkGray">10.0/16 local</span> 
~~~ 

This rules only recognises network packets with source or destination addresses in the 10.0.0/24 subnet and enables connections through the 'local' route. Any packets with source or destination addresses outside the 10.0/16 subnet will be dropped.