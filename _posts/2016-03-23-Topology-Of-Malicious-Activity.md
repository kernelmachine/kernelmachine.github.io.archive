---
layout: post
title: The Topology of Malicious Activity on IPv4
comments: true
---

At Rapid7, we're building tools that help us investigate the threat landscape across the Internet. Projects Sonar[^1] and Heisenberg[^2] give us global exposure to common vulnerabilities and patterns in offensive attacks. Our machine learning projects can detect and characterize phishing URLs and SSL certificates. Our threat intelligence repository is growing with datasets that resolve malicious activity to address blocks and autonomous systems.

We have recently focused our research on how these tools can work together to provide unique insights on the state of the Internet. In this post, we'll present the beginning of our explorations to identify stable, macro-level attacks trends invisible on the scale of IP addresses.

### IPv4 Topology
First, a quick primer on IPv4, the fourth version of the Internet Protocol. The topology of IPv4 is characterized by three levels of hierarchy, from smallest to largest: IP addresses, subnets, and autonomous systems (ASes). IP addresses on IPv4 are 32-bit sequences that identify hosts or network interfaces. Subnets are groups of IP addresses, and ASes are blocks of subnets managed by public institutions and private enterprises. IPv4 is divided into about 65,000 ASes, at least 30M subnets, and $$2^{32}$$ IP addresses.

![Phishing Activity](http://pegasos1.github.io/public/20160215/fig1.png)


### Malicious ASes
There has been a great deal of academic and industry focus on identifying malicious activity across
autonomous systems, and for good reasons. Over 50% of “good” Internet traffic comes
from large, ocean-like ASes pushing content from companies like Netflix, Google, Facebook, Apple and Amazon.  However, despite this centralized content, the vastness of the Internet enables those with malicious intent to stake their claim in less friendly waters. In fact, our longitudinal data on phishing activity across IPv4 presented an interesting trend: *a small subset of autonomous systems have regularly hosted a disproportionate
amount of malicious activity*. In particular, 200 ASes hosted 70% of phishing activity from 2007 to 2015
(data: cleanmx archives[^7]). We wanted to understand what makes some autonomous systems more
likely to host malicious activity.

![IPv4 fragmentation](http://pegasos1.github.io/public/20160215/fig2.png)




### IPv4 Fragmentation

We gathered historical mappings between IP addresses and ASes from 2007 to 2015 to understand the history of IPv4. The data clearly suggested that IPv4 has been fragmenting. The total number of ASes has grown about 60% in the past decade, and during the same period, there has been a rise in the number of small ASes and a decline in the number of large ones. These results make sense given that the IPV4 address space has been exhausted. This means that growth in IPv4 access requires the reallocation of existing address space into smaller and smaller independent blocks.

![AS fragmentation](http://pegasos1.github.io/public/20160215/fig3.png)



### AS Fragmentation

Digging deeper into the Internet's topological hierarchy, we analyzed the composition, size, and fragmentation of malicious ASes.

ARIN, one of the primary registrars of ASes, categorizes subnets based on the number of IP addresses they contain.[^8] We found that the smallest subnets available made up on average 56 $$\pm$$ 3.0 percent of a malicious AS.

Furthermore, we inferred the the size of an AS by calculating its maximum amount of addressable space. Malicious ASes were in the 80-90th percentile in size across IPv4.  

Finally, to compute fragmentation, subnets observed in ASes overtime were organized into trees based on parent-child relationships (Figure 3). We then calculated the ratio of the number of root subnets, which have no parents, to the number of child subnets across the lifetime of the AS. We found that malicious ASes were 10-20% more fragmented than other ASes in IPv4.

These results suggest that malicious ASes are large and deeply fragmented into small subnets. ARIN fee schedules showed that smaller subnets are significantly less expensive to purchase.[^8] Cheap, small subnets may allow malicious registrars to purchase many IP blocks for traffic redirection or proxy servers, so they can better float under the radar.


![topologies](http://pegasos1.github.io/public/20160215/fig5.png)





### Conclusion

Our research presents the following results:

 1) A small subset of ASes host a disproportionate amount of malicious activity.

 2) Smaller subnets and ASes are becoming more ubiquitous in IPv4.

 3) Malicious ASes are likely large and deeply fragmented

Further work is required to characterize the exact cost structure of buying subnets, registering IP blocks, and setting up infrastructure in malicious ASes.

We'd also like to understand the network and system features that convince attackers to co-opt a specific AS over another. For example, we used Sonar’s historical forward­DNS service and our phishing detection algorithms to characterize the domains that have mapped to these ASes in the past two years. Domains hosted in malicious ASes had features that suggested deliberate use of specific infrastructure. For example, 'wordpress' sites were over-represented in some malicious ASes (like AS4808), and GoDaddy was by far the most popular registrar for malicious sites across the board. Are these features core to the design and sustenance of phishing attacks?

As seen below, it's also possible to analyze SSL certificates to uncover the distribution of devices hosted in ASes across IPv4.

![malicious infrastructure](http://pegasos1.github.io/public/20160215/fig4.png)

Each square above shows the probability distribution of device counts of a particular type. Most ASes host fewer than 100 devices across a majority of categories. Are there skews in the types of devices used to propagate phishing attacks from these malicious ASes?

This research represents an example of how Internet-scale data science can provide valuable insight on the threat landscape. We hope similar macro level research is inspired by these explorations.       



*Thanks to Bob Rudis (@hrbrmstr) for help with this post.*

[^1]:[Sonar intro](https://sonar.labs.rapid7.com/)
[^2]:[Heisenberg intro](https://community.rapid7.com/community/infosec/blog/2016/01/05/12-days-of-haxmas-beginner-threat-intelligence-with-honeypots)
[^3]:[Internet Bad Neighborhoods: The spam case](http://pegasos1.github.io)
[^4]:[FIRE: FInding Rogue nEtworks](http://pegasos1.github.io)
[^5]:[Abnormally Malicious Autonomous Systems and Their Internet Connectivity](http://pegasos1.github.io)
[^6]:[Malicious Hubs: Detecting Abnormally Malicious Autonomous Systems](http://pegasos1.github.io)
[^7]:[Cleanmx archive](http://cleanmx.org)
[^8]:[ARIN fee schedules](https://www.arin.net/fees/fee_schedule.html)
