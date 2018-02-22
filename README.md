# README #

This repo contains the CloudFormation scripts that can be used to create a resilient Cluster of Web servers (nginx).

### What is the structure ###

A VPC, that has 6 subnets
2 Public (for the Applicatin Load Balancer)
2 App (For the Instances)
2 NATs (for the Internet NAT's)

A new DNS Hosted Zone is used to contain the CNAME DNS record that points to the ALB.

The Instances are based on Ubuntu 16.04, when they start they install Nginx and replace the homepage.

### What do I need before the stack creation ###

Before you start:

On your local machine
* Install AWS CLI - https://aws.amazon.com/cli/
* run the command "aws configure --profile TestDeployment" - enter your details
* * Access Key - value from your AWS deploy user (see below)
* * Access Secret- value from your AWS deploy user (see below)
* * Region - eu-west-1 (this is your choice - this value is Ireland)
* * Format - json

On your AWS account you will need
* A new user in IAM that has the Role of "Admin", check the programatic access only option
* A DNS root Zone (optional), if you don't want to have a real DNS name then comment out the Route53 parts of the stacks
* A EC2 Key - manually created for SSH'ing onto the Instances.


### How do I create the Stacks using these scripts? ###

Bring Stack BASE and NETWORK up together, then when they have both completed bring up LOADBALANCER, then finally APP.

* From the Command Line these scripts may be used to create the stack based on the scripts included in this repo.
~~~~
*** TEST
~~~~
    aws cloudformation create-stack --stack-name test-web-base --template-body file://00-base.yaml --parameters file://params/test/base.json --profile 'TestDeployment'
    aws cloudformation update-stack --stack-name test-web-base --template-body file://00-base.yaml --parameters file://params/test/base.json --profile 'TestDeployment'
    aws cloudformation delete-stack --stack-name test-web-base --profile 'TestDeployment'
 
    aws cloudformation create-stack --stack-name test-web-network --template-body file://10-network.yaml --parameters file://params/test/network.json --profile 'TestDeployment'
    aws cloudformation update-stack --stack-name test-web-network --template-body file://10-network.yaml --parameters file://params/test/network.json --profile 'TestDeployment'
    aws cloudformation delete-stack --stack-name test-web-network --profile 'TestDeployment'
 
    aws cloudformation create-stack --stack-name test-web-loadbalancer --template-body file://20-loadbalancer.yaml --parameters file://params/test/loadbalancer.json --profile 'TestDeployment'
    aws cloudformation update-stack --stack-name test-web-loadbalancer --template-body file://20-loadbalancer.yaml --parameters file://params/test/loadbalancer.json --profile 'TestDeployment'
    aws cloudformation delete-stack --stack-name test-web-loadbalancer --profile 'TestDeployment'
 
    aws cloudformation create-stack --stack-name test-web-app --template-body file://30-app.yaml --parameters file://params/test/app.json --capabilities CAPABILITY_IAM --profile 'TestDeployment'
    aws cloudformation update-stack --stack-name test-web-app --template-body file://30-app.yaml --parameters file://params/test/app.json --capabilities CAPABILITY_IAM --profile 'TestDeployment'
    aws cloudformation delete-stack --stack-name test-web-app --profile 'TestDeployment'

* Configuration
The configuration is held in the params folder "test", you can create new folders for each environment.
You will need to chancge the configuration values so they don't clash with the other environments (if you are standing them up in the same account)

### How do I know it works

For this example configuration, you should be able to open the following page in a  browser http://www.test.cloudformationexample.co.uk
The following steps occur

* Your system using DNS asks your configured DNS Server what is the IP for : www.test.cloudformationexample.co.uk
* The recurive DNS server queries who is in control of the DNS root cloudformationexample.co.uk.
* The recurive DNS Server then contacts the Authorative DNS server and requests which DNS server is responsibel for the ZONE test.cloudformationexample.co.uk
* The recursive DNS server then asks the DNS servers who are responible for the Zone, for the Record for www.test.cloudformationexample.co.uk
* The Zone serves return the CNAME for the ALB (Load Balancer)
* The recursive then queries who is in control of the DNS root amazonaws.com.
* The recursive  then works it's way down the zones until it reaches eu-west-1.elb.amazonaws.com.
* The recursive receives the A records (2 of them) for test-web-publicalb-1111111111111.eu-west-1.elb.amazonaws.com
* The Recursive then caches this response and returns them to your system
* Your browser then chooses one of these IP's and creates a connection on port 80, the packet goes accross a number of hops on the Internet.
* The ALB (in the Public Subnet) recieves this TCP connection request and forwards it onto one of the Instances.
* The request leaves the Public subnet and is routed (by the Public Routing table) to APP Subnet
* The NGINX server on the instance revieves the request, opens a connection, and sends back a TCP packet back to your client.
* The TCP Packet leaves the APP Subnet, it is routed by the App Routing Table to the NAT Subnet.
* A NAT instance recieves the TCP traffic and forwards it on (using the NAT Routing Table) to the Internet Gateway
* The Intenet Gateway returns the TCP packet back to your System (via a number of hops on the internet)
* Your Browser then sends the GET command over the open connection (GET / HTTP/1.1; host: www.test.cloudformationexample.co.uk)
* The request goes back though the internet, ALB (public subnet), to the APP (APP Subnet)
* The Instance receives the request on port 80 and Nginx deals with the request, returning the content of the page /var/www/html/index.html
* The response is streamed back via the NAT Gateway and Inetrnet Gateway back to your Browser