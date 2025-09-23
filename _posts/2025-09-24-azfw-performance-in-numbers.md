---
layout: post
title: "Azure Firewall Performance Under the Microscope"
date: 2025-09-24
description: "A data-driven analysis comparing the performance characteristics of Azure Firewall Standard and Premium SKUs."
image: "/images/2025-09-24-logo.png"
categories: ["Azure Firewall", "Azure Bastion", "Performance"]
githubissuesid: 48
---

![logo](/images/2025-09-24-logo.png)

My customer recently asked me about the performance differences between Azure Firewall Standard and Premium SKUs. This prompted me to conduct a "sort-of" data-driven analysis comparing the performance of different SKUs.

Azure Firewall is a robust solution for protecting Azure Network infrastructure. With two primary SKUs available - Standard and Premium - understanding their performance characteristics is crucial for making informed decisions. 

In this post, I will be mainly focusing on the throughput performance and will not cover any other aspects of Azure Firewall product, like features, pricing or deployment scenarios.

## What is Azure Firewall?

Azure Firewall is cloud native, highly available, with built-in auto scaling firewall-as-a-service designed to handle large amounts of traffic. 
Azure Firewall comes in three SKUs: Basic, Standard, and Premium.

## The test plan

- On Azure the throughput between two VMs is determined by the VM size. For my test lab I used four pairs of `Standard_D2ds_v6` VMs. `Standard_D2ds_v6` is a general purpose VM size with 2 vCPUs and 8 GiB of memory. According to [Azure documentation](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/general-purpose/ddsv6-series?tabs=sizenetwork) the maximum network bandwidth for this VM size is 12.5 Gbps.
- I used `iperf3` tool to measure the throughput between VMs. `iperf3` is a widely used network testing tool that can create TCP data streams and measure the throughput of a network that is carrying them.
- Azure Firewall uses five to seven minutes to scale up. Therefore I needed to run several (five to be precise) 10 min test sessions with 5 min apart to ensure that Azure Firewall had enough time to scale up.

Here is the master-plan:

1. Implement Hub-and-Spoke network topology with Azure Firewall Standard and Azure Bastion Standard to secure the ssh sessions to the VMs.
2. Deploy four pairs of VMs in the spoke virtual networks. Each pair is represented with server and client VMs. 
3. Establish a performance baseline by measuring the throughput of a simple VM-to-VM connection bypassing Azure Firewall (use direct peering between VNets).
4. Remove direct peering between spoke VNets and route traffic through Azure Firewall Standard. 
5. Measure the throughput between VMs with Azure Firewall Standard in place. 
6. Upgrade Azure Firewall Standard to Premium SKU.
7. Measure the throughput between VMs with Azure Firewall Premium in place.

## IaC

All resources used in this test lab can be found [here](https://github.com/evgenyb/azfw-perf).
If you want to deploy this lab in your Azure subscription, please follow the instructions in the `README.md` file.

If you provision lab, please don't forget to destroy it afterwards to avoid unnecessary costs.
Both Azure Firewall (Standard and especially Premium) and Azure Bastion quite **expensive** resources, so keeping them running when not needed can lead to significant costs.

## Baseline performance without Firewall

I used `iperf3` command and run test for 10 min with 32 parallel streams:

```bash
iperf3 -c <serverIP> -t 600 -P 32
```

With direct peering between spoke VNets, the throughput between VMs was pretty much maxed out at the VM Sku limit of 12.5 Gbps.

| Test pair | Data transferred | Throughput (Gbps) |
|-----------|------------------|------------------|
| client1 -> server1 | 739 GBytes | 10.6 |
| client2 -> server2 | 859 GBytes | 12.3 |
| client3 -> server3 | 859 GBytes | 12.3 |
| client4 -> server4 | 852 GBytes | 12.2 |

Here is the printscreen of this test run:
![baseline](/images/2025-09-24-baseline.png)


## Performance with Azure Firewall Standard

First, let's collect [Standard SKU specifications](https://learn.microsoft.com/en-us/azure/firewall/firewall-performance#performance-data)

| Firewall use case | Specification |
|---------|------------------|
| Total throughput for initial firewall deployment | 3 Gbps |
| Max bandwidth | 30 Gbps |
| Max bandwidth for single TCP connection | 1.5 Gbps |
| Scale up time | 5-7 minutes |

Now, let's see how Azure Firewall Standard performs.
I removed the direct peering between spoke VNets and routed all traffic (east-west and north-south) through Azure Firewall Standard.

Then I run five 10 min `iperf3` test sessions using 32 parallel streams with 5 min apart to ensure that Azure Firewall had enough time to scale up. 

Here is the command I ran on each client VM:

```bash
iperf3 -c <serverIP> -t 600 -P 32
```

And here is the summary of the results:

### Run 1 - 4 sessions, 10min, 32 streams 

| Test pair | Throughput (Gbps) |
|-----------|------------------|
| client1 -> server1 | 1.86 |
| client3 -> server3 | 1.57 |
| client4 -> server4 | 2.66 |

![sessions-running1](/images/2025-09-24-fw-standardsessions-running1.png)
![fw-standard-run1](/images/2025-09-24-fw-standard-run1.png)

### Run 2 - 4 sessions, 10min, 32 streams

| Test pair | Throughput (Gbps) |
|-----------|------------------|
| client1 -> server1 | 2.13 |
| client2 -> server2 | 2.21 |
| client3 -> server3 | 2.55 |
| client4 -> server4 | 1.89 |

![sessions-running1](/images/2025-09-24-fw-standardsessions-running2.png)
![fw-standard-run2](/images/2025-09-24-fw-standard-run2.png)

### Run 3 - 4 sessions, 10min, 32 streams

| Test pair | Throughput (Gbps) |
|-----------|------------------|
| client1 -> server1 | 3.79 |
| client2 -> server2 | 3.52 |
| client3 -> server3 | 5.37 |
| client4 -> server4 | 3.80 |

![sessions-running1](/images/2025-09-24-fw-standardsessions-running3.png)
![fw-standard-run3](/images/2025-09-24-fw-standard-run3.png)

### Run 4 - 4 sessions, 10min, 32 streams

| Test pair | Throughput (Gbps) |
|-----------|------------------|
| client1 -> server1 | 4.33 |
| client2 -> server2 | 7.25 |
| client3 -> server3 | 7.88 |
| client4 -> server4 | 12.2 |

![sessions-running1](/images/2025-09-24-fw-standardsessions-running4.png)
![fw-standard-run4](/images/2025-09-24-fw-standard-run4.png)

### Run 5 - 4 sessions, 10min, 32 streams

| Test pair | Throughput (Gbps) |
|-----------|------------------|
| client1 -> server1 | 7.69 |
| client2 -> server2 | 6.33 |
| client3 -> server3 | 6.40 |
| client4 -> server4 | 7.23 |

![sessions-running1](/images/2025-09-24-fw-standardsessions-running5.png)
![fw-standard-run5](/images/2025-09-24-fw-standard-run5.png)

### Results interpretation

As we can see from the results above, Azure Firewall took some time to scale up. During the first test session the throughput was ranging from 1.57 to 2.66 Gbps. That confirms to the `Total throughput for initial firewall deployment` specification of 3 Gbps.
During the second session the throughput scaled up to 8 Gbps, third session - to 17 Gbps, fourth session - to 25 Gbps, and finally the fifth session the throughput was almost maxed out at Standard SKU limit of 30 Gbps.


With that - thanks for reading!
