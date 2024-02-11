+++
title = 'Trying out AWS Network Reachability Analyzer'
date = 2023-04-08
draft = false
tags = [
    "aws",
    "cloud-networking",
    "operations"
    ]
toc = false
+++

This is a tool that I have seen being mentioned in blog posts and Youtube short tutorials but have never got a chance to work on. So I decided to give it a try and blog my discovery process.

#### What is AWS Reachability Analyzer?

AWS Reachability Analyer is an AWS native tool that enables you to test network connectivity between a source resource and a destination resource in your VPCs. Contrary to what I initially thought, this tool doesn't actually generate any traffic between the source and destination to test connectivity (ping/traceroute testing came to my networking mind right of the bat).

#### How does this work?

This tool seems to work by checking the configurations of all the network and security constructs (route tables, security groups, NACLs etc) chained together between the source and destination to build a model of the network and then analyze it to see if the network connectivity would work or not. This should immediately make you think that there are limitations to this, such as cases where the source and destinations are not part of the same account or organization, and this results in permission issues. More on this later. Another limitation is that this can check for only TCP and UDP traffic and not ICMP.

#### My test setup.

Since I was very interested to test how deep can this tool dig into the network connectivity, I wanted to go slightly complicated on the connectivity. As shown below, I have two VPCs in two different accounts (PROD and DEV). I also have another account (GENERAL) which is the 'delegated administrator' account, and all these three accounts belong to one AWS Organization. This setup helps to easily monitor connectivity across multiple accounts that are part of the organization from one 'delegated administrator' account.

![Topology](/images/2023-04-08-image01.png)

In each of these VPCs I have a public subnet (only so that I can SSH into the instances) with a test instance deployed in them. I have a Transit Gateway deployed in the PROD account which I then shared with the DEV account. The TGW is then attached to the private subnets in the respective VPCs in both the accounts. The reason to choose the private subnet instead of the public subnet is explained [here](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-nacls.html) where the NACLs are ignored in certain situations if the TGW attachment and instances are in the same subnet. With some basic routing, I have connectivity enabled between the two instances.

When everything is setup right and I have the right credentials and keys, I can successfully SSH from PROD instance to the DEV instance using the private address, so that the traffic traverses within our AWS setup and not over the internet.

{{< highlight bash >}}
[ec2-user@ip-10-1-0-10 ~]$ ssh -i "saac03-dev-usea1.pem" ec2-user@10.2.0.10
   ,     #_
   ~\_  ####_        Amazon Linux 2023
  ~~  \_#####\
  ~~     \###|
  ~~       \#/ ___   https://aws.amazon.com/linux/amazon-linux-2023
   ~~       V~' '->
    ~~~         /
      ~~._.   _/
         _/ _/
       _/m/
Last login: Thu Apr 6 13:02:30 2023 from 10.1.0.10
[ec2-user@ip-10-2-0-10 ~]$
{{< /highlight >}}

With a working setup, when I run the reachability analyzer from the GENERAL account, I need to mention details such what are the source and destination accounts, what kind of resources are we testing connectivity between (instances in my case), optionally the source and/or destination port and whether the traffic is TCP or UDP. The tool takes a few seconds and shows you the result of its findings. Below is what a successful path looks like. There is a lot of details in one place for easy visibility and troubleshooting as we can see  when I purposefully break this path.

![Working Path Analysis](/images/2023-04-08-image02.png)

That looks clean! Now let's break something and see how the results come back. I am going to delete the route towards the DEV VPC in the PROD VPC TGW route table. This means the DEV VPC has route to PROD VPC via the TGW but the PROD does not have a route to reach the DEV VPC. Let's see what the Network Reachability Analyzer finds.

![Broken Path Analysis](/images/2023-04-08-image03.png)

If you notice the red icon in the screenshot above, you can notice that the tool has discovered the missing route in the exact TGW route table, just as expected. If you click on the TGW route table name, it shows you further details that will help us login to the right account and navigate straight to the entity with the problem and fix it.

There is a LOT of specific details in one place for easy visibility and troubleshooting as we can see  when I purposefully break this path. I remember the time when I used to this manually by hopping back and forth over multiple pages and sections for troubleshooting.

I have to point out that this didn't work as well when I experimented with breaking a few other things such as when I blocked the SSH on the NACL in the DEV account. The tool did realize the communication was broken but showed me the wrong explanation and also keep in mind this is not free! There is a charge per "path analysis".
