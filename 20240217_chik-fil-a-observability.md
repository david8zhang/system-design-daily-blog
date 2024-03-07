# How Chick-Fil-A Manages Thousands of Kubernetes Kubernuggets

_by David Zhang, published Feb 17th, 2024_

"Rev up those fryers, cause I am sure hungry for some Edge-Compute Observability!"

## Background Info

Back in 2018, Chick-Fil-A was suffering from success - customers were flocking there due to its high quality and unmatched customer service. It wasn't uncommon to see lines going out the door. Many of their busiest restaurants were handling over three times the volume they were designed for.

They had a capacity problem, and they needed to scale up fast.

As it turns out, _technology_ was the answer. It was time to Chick-Fil-Allocate some Kentucky Fried Clusters (I'll be here all week, ladies and gentlemen)

## Let Him Cook

Like me, you might be thinking that this sounds...a bit wacky. A chicken restaurant rolling out a Kubernetes control plane for thousands of mini-clusters distributed across physical restaurant locations? What's next, Multi-Feeder Replication to ensure fault-tolerance at your chicken farms? You gonna shard those cripsy chicken strips using consistent hash-browning? (Gettin' kinda hungry now...)

But as I kept reading their engineering blogs, I began to eat what Chick-Fil-A was feeding me, both figuratively and literally.

The initial use case was all about forecasting demand. Initially, Chick-Fil-A envisioned collecting telemetry data from point of sale systems to intelligently forecast how much of a certain menu item should be prepared in advance. For example, using some fancy analytical queries that look at transaction-level sales data, they can predict that five thousand grilled club sandwiches should be cooked on Tuesdays. (Not real data)

But as it turns out, the Chick-Fil-A engineering team is living in the year 3024. They decided they wanted to get even saucier with their data pipeline by outfitting kitchen equipment with IoT sensors to monitor operational information about the actual order preparation process.

This sensor data can be fed into an analytics engine, which can drive a real-time dashboard indicating what restaurant team members should cook at any given time. This data can also drive future cooking automation initiatives.

## Chick-Fil-A's Chick-Fil-Architecture for High Chick-Fil-Availability

From the outset, Chick-Fil-A wanted free range computing. Unlike their chicken feed, they didn't want to be siloed in by the incumbent cloud providers. As a result, they rolled out their own open-ecosystem data platform.

In practical terms, their architecture is split into 2 pieces: First, they have a cloud-based control plane which performs things like authorization, data ingest and routing, deployment management, and time series data processing for operational purposes (logging, monitoring, alerting, and all that good stuff).

Second, they maintain a huge edge-computing ecosystem, in which every restaurant literally runs and maintains their own little server rack on commodity hardware (specifically, [consumer grade Intel NUCs](https://en.wikipedia.org/wiki/Next_Unit_of_Computing)). Edge compute workloads include things like processing data from IoT sensors, running ML workloads for forecasting purposes, and handling log collection, monitoring, and pub/sub messaging.

## The Edgiest Compute Platform

Let's dive into that Edge Compute piece (which I and CFA themselves seem to think is the coolest part of their architecture).

Chick-Fil-A's setup is unique in that instead of having a few giant clusters with hundreds of thousands of containers each, they have a few thousand clusters with less than a hundred containers each. This is a bit of a problem, since most existing K8s tools and services are meant for large-scale data-center deployments. As a result, they had to cook some stuff up in house.

One thing they did was use K3s instead of K8s in their edge clusters. K3s is an open-source version of Kubernetes that's more lightweight and less resource-intensive. It doesn't have any of the extra cloud features that aren't applicable to edge computing environments. It's also about 3 billion lines of code leaner than its K8s counterpart and can run on less than 512 MB of RAM.

They also use a nifty tool called Vector, an observability data pipeline provided by Datadog for routing, filtering, and forwarding logs. One issue they wrangled with was when an IoT sensor went bonkers and started blasting the cloud control plane with log data, devouring the bandwidth of the local restaurant network and blocking credit card transactions from going through. As a result, they set up Vector like a surge protector, passing all logs from IoT producers through its rules and filtering logic before sending them off to the cloud. Vector also provided some added benefits in terms of abstracting log consumers from producers, and giving the engineering team finer-grained flow control for their log streams.

One last thing worth mentioning is their [GitOps](https://www.redhat.com/en/topics/devops/what-is-gitops) deployment management system. Every restaurant maintains its own Git repo that maintains their infrastructure as code configuration file. An in-house agent called Vessel polls their self-hosted GitLab instance in the cloud to keep their local repos in sync. A new deployment ammounts to a change being merged onto the master branch of these repos. For multi-cluster deployments, they rolled out a Deployment Orchestration API which talks to Feedback Agents running on each cluster to track deployment statuses on the cloud.

![Chick-fil-a](https://firebasestorage.googleapis.com/v0/b/system-design-daily.appspot.com/o/Chickfila-architecture.png?alt=media&token=4deef5fc-863a-4962-89fc-cc19dc27a2a9)

## Conclusion

I guess as the old saying goes, if it looks like a chicken and clucks like a chicken, it might just be a tech company. It's almost as if each Chick-Fil-A restaurant is a separate service in a giant web of physical microservices. They each maintain their own edge compute clusters, with standardized logging and deployment processes managed by a central cloud coordination layer. Instead of serving HTTP requests they serve up grilled nuggets and waffle fries.

So the next time you're scarfing down a spicy deluxe sandwich with a side of mac and cheese, think about the incredible feats of engineering that went in to making that meal happen.

Original blog posts and tech talks:

- [Edge Computing at Chick-Fil-A](https://medium.com/chick-fil-atech/edge-computing-at-chick-fil-a-2621f4b5a969)
- [Enterprise Restaurant Compute](https://medium.com/chick-fil-atech/enterprise-restaurant-compute-f5e2fd63d20f)
- [The Edge of Observability: How Chick-fil-A Observes a Fleet of 2800 Restaurant-deployed K8s Clusters](https://www.youtube.com/watch?v=W1HKFIb_2hs&t=59s)
