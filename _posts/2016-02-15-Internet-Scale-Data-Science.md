---
layout: post
title: Internet-Scale Data Science
comments: true
---

At Rapid7, we're building tools that help us investigate the threat landscape across the Internet. We're working on Projects Sonar[^1] and Heisenberg[^2] which give us global exposure to common vulnerabilities and patterns in offensive attacks. Our nascent machine learning projects can detect and characterize phishing URLs and SSL certificates. Our threat intelligence repository is growing with datasets that resolve malicious activity to address blocks and autonomous systems.

We have recently focused our research on how these tools can work together to provide unique insight into the state of the Internet. The analysis of the Internet in its entirety can allow researchers to identify stable trends in the threat landscape that are emergent properties of seemingly disorganized, random patterns in individual attacks between IP addresses. In this post, we'll present the beginning of these explorations.

### IPv4 Topology
First, a quick primer on IPv4, the fourth version of the Internet Protocol. The topology of IPv4 is characterized by three levels of hierarchy: IP addresses, subnets, and autonomous systems (ASes). IP addresses on IPv4 are 32-bit sequences that identify hosts or network interfaces. Subnets are groups of IP addresses, and ASes are blocks of subnets managed by public institutions and private enterprises. IPv4 is divided into about 65,000 ASes, 200,000,000 subnets, and 2^32 IP addresses.

### IPv4 Sinkholes

Longitudinal data on phishing activity across IPv4 presented an interesting trend: *a small subset of autonomous systems have hosted a disproportionate amount of malicious activity.* In particular, 200 ASes hosted 70% of phishing activity from 2007 to 2015 (data: cleanmx archives). We wanted to understand what makes some autonomous systems more likely to host malicious activity.


![Phishing Activity](http://pegasos1.github.io/public/20160215/fig1.png)


### IPv4 Fragmentation

We gathered historical data on the mapping between IP addresses and ASes from 2007 to 2015 to generate a longitudinal map of IPv4. This map clearly suggested that IPv4 has been fragmenting. Since 2005, the total number of ASes has been growing about 70% per year. During the same period, there has been a rise in the number of small ASes and a decline in the number of large ones. These results make sense given that IPV4 address space is finite and has been exhausted. This means that growth in IPv4 access requires the reallocation of existing address space into smaller and smaller blocks.

![IPv4 fragmentation](http://pegasos1.github.io/public/20160215/fig2.png)

### AS Fragmentation
Digging deeper into the Internet hierarchy, we analyzed the composition, size, and fragmentation of ASes that have hosted disproportionate amounts of phishing activity since 2007. We wanted to analyze the distribution of subnets that made up these ASes.

ARIN, one of the primary registrars of ASes, categorizes subnets based on the number of IP addresses they contain. We found that xx-small subnets, the smallest subnets available, made up 56 $$\pm$$ 3.0 percent of a malicious AS. ARIN fee schedules showed that smaller subnets are significantly less expensive to purchase.

The size of an AS is inferred by calculating the maximum addressable space of the AS. Malicious ASes were in the 80-90th percentile in size across IPv4.  

To compute fragmentation, subnets observed in ASes overtime were organized into trees based on the parent-child relationships observed (Figure 3). We then calculated the ratio of the number of root subnets, which have no parents, to the number of subsequent child subnets across the lifetime of the AS. We found that malicious ASes were 10-20% more fragmented than other ASes in IPv4.

![AS fragmentation](http://pegasos1.github.io/public/20160215/fig3.png)

These results suggest that malicious ASes are large and deeply fragmented into small subnets.


### Malicious Infrastructure

We wanted to characterize the distribution of devices and infrastructure used across malicious ASes. We  used Sonar's historical forward-DNS service and our automated phishing detection tool to characterize other domains that have mapped to these ASes. There were concentrated lexical and server features of hosted domains, like the fact that 'wordpress' sites were over-represented, or that GoDaddy was by far the most popular registrar for these malicious sites. Using our automated certificate classification tool, we found that some malicious ASes were more likely to use specific devices, suggesting a deliberate use of malicious infrastructure, perhaps based on cost structure or end goals.

![malicious infrastructure](http://pegasos1.github.io/public/20160215/fig4.png)

### Conclusion

 Our research presents the following results:

  1) There exist sinkholes in IPv4 -- a small subset of ASes that host a disproportionate amount of malicious activity.

  2) Smaller subnets and ASes are becoming more ubiquitous in IPv4.

  3) Malicious ASes are likely large and deeply fragmented

  4) There is a concentrated use of specific infrastructure in malicious ASes


Further work is required to characterize the exact cost structure of buying subnets, registering autonomous systems, and setting up malicious infrastructure. This work would help us further understand why these trends in macro-scale malicious activity exist.

This research presents our team's first foray into the integration of many powerful Rapid7 tools to do Internet-scale data science. We hope similar large-scale research is inspired by these explorations.       


[^1]: (https://sonar.labs.rapid7.com/)
[^2]:(https://community.rapid7.com/community/infosec/blog/2016/01/05/12-days-of-haxmas-beginner-threat-intelligence-with-honeypots)
[^3]: [History of the Internet]
[^4]: [Cleanmx archive] (http://cleanmx.org)
