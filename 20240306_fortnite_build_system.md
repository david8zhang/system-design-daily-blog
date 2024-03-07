# How Epic Games Builds Fortnite in Less than a Fortnite

_by David Zhang, published Feb 17th, 2024_

"How Epic Games builds Fortnite so you can build in Fortnite."

## Background Info

Epic Games was going off in 2020, as Fortnite was huge and only getting bigger - both in terms of playerbase and codebase size. That meant that the engineers as Epic had to figure out a reliable, fast way to compile and test playable game builds. This turned out to be trickier than it sounds, since another thing was also going off in 2020 - the COVID-19 pandemic.

Epic found themselves needing to deliver fast builds for a now globally distributed, remote workforce of engineers and QA testers. Unfortunately, their legacy on-prem infrastructure made things hard. Builds took forever - some days there wouldn't be any testable build at all. Furthermore, their test coverage was in the toilet and breaking changes were hard to track down. They needed scalability, performance and automation. And they needed it NOW.

## Taking Their On-Prem Datacenter and Pushing It Somewhere Else.

![patrick-push](https://firebasestorage.googleapis.com/v0/b/system-design-daily.appspot.com/o/patrick-push-meme.jpeg?alt=media&token=564f3ded-71b3-4749-b8c9-b42f7ddba4e6)

Epic made a deal with the devil and began to migrate their build system to AWS. They started by setting up a VPC and a Windows builder instance manually in the AWS console. After verifying that that worked, they decided to automate instance provisioning using some common IAC / DevOps tools:

- [Ansible](https://www.ansible.com/) for configuration management
- [Packer](https://www.packer.io/) for creating images with pre-baked configuration
- [Terraform](https://www.terraform.io/) for codifying the rest of the infra, auto-scaling groups, VPCs, subnets, route-tables, etc.

This enabled them to quickly spin up new builder instances, providing their system with faster scaling capabilities.

But what about performance? There was a big issue on that front - all their dedicated storage was still on prem.

To build Fortnite, they had two main types of data stores:

1. Derived data cache, which stores intermediate states produced by the Unreal Engine "cook" process, or the process by which content is converted into different formats for various platforms.

2. Temp storage

Their derived datacache uses the Server Message Block (SMB) file sharing system, which is a client-server communication protocol for sharing access to files (and other resources like serial ports and printers) on a network. Historically, it's been used for connecting Windows systems, but Linux and macOS also include client components for accessing SMB resources.

As for temp storage, they were using a standard Network-Attached Storage (NAS) system. (NAS is basically just a computer with a bunch of hard drives connected to your network that you can use to store files and other data)

They decided to just take all this stuff and shove it into the Cloud. Specifically, they used [AWS FSx](https://docs.aws.amazon.com/fsx/latest/WindowsGuide/what-is.html), a fully managed file storage system built on Windows Server. This was pretty much a no-brainer given they were already using SMB with Windows for DDC anyways.

Their version control system ran on Perforce on-prem. They decided to yeet that into the cloud as well, deploying and scaling it using EC2 and [EBS](https://aws.amazon.com/ebs/).

Finally, they migrated their build system to the cloud. Their build system used [Incredibuild](https://www.incredibuild.com/), a distributed compile system that enables distribution and scheduling of workloads across multiple cores on multiple machines.

Rome wasn't built in a day, of course. Facillitating their gradual transition required them to really widen their network pipes - they did so by increasing their direct connect bandwidth betwen their AWS VPC and their on-prem private cloud from 10 to 40GBps

## Arrgh, Cloud Migration, You're Spending All Me Money!

Copy pasting their on-prem setup into the cloud ended up costing quite a lot. Their "lift and shift" approach meant that their infra was mostly statically allocated and always on. That means resources would often get underutilized.

Their initial answer to this was to try to use super dynamic infrastructure with ephemeral or containerized builders. But spinning up instances to build the massive Fortnite codebase surfaced some issues with data mobility.

In short, every time they spin up these new instances they need to sync data from somewhere. That's just not feasible, so they opted for a simple "power on/power off" scaling approach instead. As the name implies, they literally just persist data on the build instances and power them off. Then when they need to build, they just power em back on.

In order to make sure the data caches are warm, they have a lambda function run periodically to sync data into the machines and warm the cache.

Incredibuild also came in clutch and introduced a "cloud cores" feature that enabled dynamic resource provisioning across multiple availability zones. It uses EC2 spot-instances underneath the hood with on-demand instances as fallbacks.

## Was It Worth It?

At the end of this migration, Epic was able to realize a number of benefits.

### Cross Platform Testing

Their original on-prem setup only enabled them to test on Windows and Mac, with AWS they were able to test across multiple different types of hardware platforms, CPU architectures, and operating systems. (e.g. GPU instances with Nvidia, AMD, CPU instances across Intel and AMD, Ubuntu instances, etc.). This helped with test coverage and enabled them to support all kinds of different configurations and platforms.

### Build Distribution

They also managed to leverage S3 multi-region replication to globally distribute testable builds of Fortnite. This meant that QA engineers worldwide didn't have to wait forever to download builds from some server super far away.

### Speed

CI frequency went from every 20 minutes to every new change. With all their fancy new builder instances, they were able to produce 10+ healthy builds per day (up from 0 to 1 build per day). Most delicious of all, they were able to reduce compile times by 75% (reducing client compile times by over 2 hours per day!) and cook times by 20%. As a result of their globally distributed S3 build replication, they were able to reduce QA build download times from 2 hours to 20 minutes.

## What Did We Learn?

Is the cloud the answer? If you have as much money as Epic Games does, it honestly might be. Though this all feels like on big ad for moving to AWS, the results are undeniable. Just like in the game itself, build speed is all that matters.
