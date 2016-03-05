---
layout: post
title: Internet-Scale Data Science
comments: true
---

At Rapid7, we are building tools that give us insight into the threat landscape across the Internet. We are working on Projects Sonar[^1] and Heisenberg[^2] which give us global exposure to common vulnerabilities and patterns in offensive attacks. Our nascent machine learning projects can detect and characterize malicious URLs and certificates. Our threat intelligence repository is growing with datasets that resolve malicious activity to address blocks and autonomous systems.

While these tools have provided value independently, we have recently focused on using these tools in concert to give unique perspectives on the state of the Internet.


### IPv4 Sinkholes

A first step in this direction was borne out of an effort to understand the following observation we made from our threat intelligence data: *A small subset of autonomous systems host a disproportionate amount of malicious activity.*

In particular, $$200$$ Autonomous systems hosted $$70%$$ of phishing activity from 2007 to 2015, and $$100$$ Autonomous systems hosted $$70%$$ of malicious password attempts on Heisenberg honeypots in 2015.

*FIGURE*

We wanted to understand what makes some autonomous systems more likely to host malicious activity.

### IPv4 Topology Primer
The topology of IPv4 is characterized by three levels of hierarchy, IP addresses, subnets, and ASes. IP addresses on IPv4 are 32-bit sequences that identify hosts or network interfaces. Subnets are groups of IP addresses. Autonomous systems (ASes) are managed blocks of subnets that have independent routing policies. IPv4 is divided into about 65,000 of these ASes, which are managed by universities, public institutions, and private enterprises. There are 200,000,000 subnets on IPv4 today, and 2^32 IP addresses.


### IPv4 Fragementation
We gathered historical data on the mapping between IP addresses and ASes from 2007 to 2015 to generate a longitudinal map of IPv4.


We found that IPv4 is fragmenting.

*FIGURE*

This trend is clear in the data. Since 2005, the total number of ASes has been growing about 70% per year. During the same period, we see a rise in the number of small ASes and a decline in the number of large ones. These results make sense given the fact that growth in IPv4 access requires the reallocation of existing address space.

### AS Fragmentation
Digging deeper, we analyzed the composition, size, and fragmentation of ASes that have hosted disproportionate amounts of phishing activity since 2007.

We found that xx-small subnets made up 56 $$\pm$$ 3.0 percent of a malicious AS.

Furthermore, malicious ASes were in the 80-90th percentile in size.

Lastly, we wanted to analyze how fragmented malicious ASes were. To compute fragmentation, subnets observed in ASes overtime were organized into trees based on the parent-child relationships observed (Figure 3). We then calculated the ratio of the number of root subnets, which have no parents, to the number of subsequent child subnets across the lifetime of the AS. We found that malicious ASes were 10-20% more fragmented than other ASes in IPv4.

*FIGURE*

These results suggest that malicious ASes are large and deeply fragmented into small subnets.

### Other Host Features
We used other tools in the OCTO arsenal to further characterize these ASes. Our AS WHOIS parser showed no particular skew in the location, time, or ownership of AS registration, suggesting that these ASes originate from similar distributions as the rest of IPv4.  We also used Sonar's historical forward-DNS service and our automated phishing detection tools to characterize other domains that have mapped to these ASes. There were concentrated lexical and server features of hosted domains, like the fact that 'wordpress' sites were over-represented, or that GoDaddy was by far the most popular registrar for these malicious sites.

### Malicious Infrastructure

Lastly, we wanted to characterize the distribution of devices used across malicious ASes. Using our automated certificate classification tool, we found that malicious autonomous systems were skewed to use specific devices, suggesting a deliberate use of malicious infrastructure, perhaps based on cost structure or end goals.

*FIGURE*

### Conclusion

 this research presents the following results:
  1) There exist sinkholes in IPv4 -- a small subset of ASes that host a disproportionate amount of malicious activity.
  2) Smaller subnets and ASes are becoming more ubiquitous in IPv4.
  3) Malicious ASes are likely large and deeply fragmented
  4) There is a concentrated use of specific infrastructure in malicious ASes
  5) The origin of these ASes are diverse

Futher work is required to characterize the exact cost structure of buying subnets, registering autonomous systems, and setting up malicious infrastructure. This work would help us further understand why these trends in macro-scale malicious activity exist.

This research presents our team's first foray into the integration of many powerful Rapid7 tools to do interesting Internet-scale data science. We hope that this type of research inspires further analysis of similar flavor.






[^1]: (https://sonar.labs.rapid7.com/)
[^2]:(https://community.rapid7.com/community/infosec/blog/2016/01/05/12-days-of-haxmas-beginner-threat-intelligence-with-honeypots)
[^3]: [History of the Internet]
[^4]: [Cleanmx archive] (http://cleanmx.org)
