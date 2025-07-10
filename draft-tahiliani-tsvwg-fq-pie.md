---
title: "Flow Queue PIE: A Hybrid Packet Scheduler and Active Queue Management Algorithm"
abbrev: "FQ-PIE"
category: exp

docname: draft-tahiliani-tsvwg-fq-pie-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Transport and Services Working Group"
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
  group: "Transport and Services Working Group"
  type: "Working Group"
  mail: "tsvwg@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/tsvwg/"
  github: "mohittahiliani/draft-tahiliani-tsvwg-fq-pie"
  latest: "https://mohittahiliani.github.io/draft-tahiliani-tsvwg-fq-pie/draft-tahiliani-tsvwg-fq-pie.html"

author:
 -
    ins: M. P. Tahiliani
    name: Mohit P. Tahiliani
    org: National Institute of Technology Karnataka
    street: P. O. Srinivasnagar, Surathkal
    city: Mangalore, Karnataka - 575025
    country: India
    email: tahiliani@nitk.edu.in
    uri: http://tahiliani.in

normative:
informative:
  LINUX-FQ-PIE:
    target: https://ieeexplore.ieee.org/abstract/document/9000684
    title: 'FQ-PIE Queue Discipline in the Linux Kernel: Design, Implementation and Challenges'
    author:
    - name: Gautam Ramakrishnan
    - name: Mohit Bhasi
    - name: V. Saicharan
    - name: Leslie Monis
    - name: Sachin D. Patil
    - name: Mohit P. Tahiliani
    date: 2019-10
    seriesinfo:
      2019 IEEE 44th LCN Symposium on Emerging Topics in Networking (LCN Symposium)
  FREEBSD-FQ-PIE:
    target: https://web.archive.org/web/20241018123533/http://caia.swin.edu.au/reports/160418A/CAIA-TR-160418A.pdf
    title: 'Dummynet AQM v0. 2–CoDel, FQ-CoDel, PIE and FQ-PIE for FreeBSD’s ipfw/dummynet Framework'
    author:
    - name: Rasool Al-Saadi
    - name: Grenville Armitage
    date: 2016-10
    seriesinfo:
      Centre for Advanced Internet Architectures, Swinburne University of Technology, Melbourne, Australia, Tech. Rep. A, 160418
  ns-3-FQ-PIE:
    target: https://www.nsnam.org/docs/models/html/fq-pie.html
    title: 'FQ-PIE Queue Discipline in ns-3'
    date: 2021-07
  REVISIT-PIE:
    target: https://www.sciencedirect.com/science/article/pii/S1389128619313775
    title: 'Revisiting Design Choices in Queue Disciplines: The PIE Case'
    author:
    - name: Pasquale Imputato
    - name: Stefano Avallone
    - name: Mohit P. Tahiliani
    - name: Gautam Ramakrishnan
    date: 2020-04
    seriesinfo:
      Computer Networks
  I-D.draft-ietf-ccwg-bbr:

--- abstract

This document presents Flow Queue Proportional Integral controller Enhanced (FQ-PIE), a hybrid packet scheduler and Active Queue Management (AQM) algorithm to isolate flows and tackle the problem of bufferbloat. FQ-PIE uses hashing to classify incoming packets into different queues and provide flow isolation. Packets are dequeued by using a variant of the round robin scheduler. Each such flow is managed by the PIE algorithm to maintain high link utilization while controlling the queue delay to a target value.

--- middle

# Introduction

Flow Queue Proportional Integral Controller Enhanced (FQ-PIE) combines flow queuing with the PIE (Proportional Integral controller Enhanced) {{!RFC8033}} Active Queue Management (AQM) algorithm to provide flow isolation and reduce bufferbloat by controlling queue delay. This is similar to how Flow Queue Controlled Delay (FQ-CoDel) {{!RFC8290}} integrates flow queuing with the CoDel (Controlled Delay) AQM algorithm {{!RFC8289}}.

When a packet is enqueued, it is classified into different queues to ensure isolation between flows. While the goal of flow queuing is to assign a unique queue to each flow, flows can instead be hashed into a set of buckets using a hash function, where each bucket corresponds to its own queue. The PIE AQM operates independently on each of these queues, enabling each flow to receive appropriate congestion signals either implicitly (via packet drops) or explicitly (via mechanisms such as Explicit Congestion Notification (ECN) {{!RFC3168}}). For dequeuing, FQ-PIE employs the Deficit Round Robin (DRR) based scheduler described in {{!RFC8290}}, which ensures fair packet scheduling across the different queues.

An implementation of FQ-PIE has been incorporated into the mainline Linux kernel as a queuing discipline (qdisc) [LINUX-FQ-PIE] and is supported by several Linux distributions. Another implementation has also been incorporated into FreeBSD [FREEBSD-FQ-PIE]. Finally, an implementation of FQ-PIE is also available in the ns-3 network simulator [ns-3-FQ-PIE].

# Terminology

This document uses the terms defined in Section 1.1 of {{!RFC8290}} and Sections 4 and 5 of {{!RFC8033}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# The FQ-PIE Algorithm

The FQ-PIE algorithm consists of two main components: (i) flow queuing, which isolates competing flows by treating flows that build queues differently from those that do not, and (ii) the PIE AQM algorithm, which manages each queue and maintains a target queue delay (recommended as 15 ms in {{!RFC8033}}). Flow queuing works by classifying incoming packets into different queues during the enqueue phase and then scheduling outgoing packets from these queues during the dequeue phase. The PIE algorithm, however, only operates during enqueue.

The details of flow queuing and the PIE algorithm are not covered here; for more information, please refer to {{!RFC8290}} and {{!RFC8033}}, respectively.

## Enqueue

The packet enqueue process is described as follows: first, the incoming packets are classified into different queues by hashing the 5-tuple, which includes the protocol number, source and destination IP addresses, and source and destination port numbers, similar to the approach used in FQ-CoDel.

Next, the packet is passed to the PIE algorithm, which uses a drop probability to determine whether the packet should be enqueued or dropped, as outlined in {{!RFC8033}}. This drop probability is updated periodically (every 15 ms, as per {{!RFC8033}}) based on the current queue delay’s deviation from the target delay and whether the delay is trending up or down.

{{!RFC8033}} presents two methods for calculating the current queue delay: one uses Little’s Law, estimating delay based on the queue length and the average dequeue rate; the other takes direct measurements using timestamps, as implemented in CoDel and FQ-CoDel. However, experimental studies on the PIE algorithm [REVISIT-PIE] indicate that while the dequeue rate is intended to estimate the transmission rate of packets over the outgoing link, it may instead reflect the rate at which packets move from the host stack (e.g., Linux qdisc) to the device driver’s transmission ring. Additionally, in FQ-PIE, queue delay estimates from Little’s Law can be unreliable, as it’s challenging to calculate an accurate per-queue dequeue rate. Consequently, the FQ-PIE algorithm SHOULD calculate the current queue delay using direct measurements with timestamps.

It is important to note that the timestamping approach provides a "per-packet queue delay," while the drop probability is calculated periodically (every 15 ms, as specified in {{!RFC8033}}). Therefore, the FQ-PIE algorithm MAY use the queue delay value from the most recently dequeued packet when calculating the drop probability.

At the time of writing this document, the Linux, FreeBSD and ns-3 implementations use timestamps to calculate the current queue delay and consider the measurements from the most recently dequeued packet when calculating the drop probability. Additionally, these implementations offer an option to use the dequeue rate estimation technique based on Little’s Law.

Lastly, if an incoming packet arrives when the total number of enqueued packets has already saturated the queue capacity, FQ-PIE drops the packet without further processing. In contrast, FQ-CoDel identifies the queue with the largest current byte count (i.e., a "fat flow") when the queue capacity is saturated and drops half of the packets from this queue (up to a maximum of 64 packets, as specified in Section 4.1 of {{!RFC8290}}). FQ-PIE does not adopt this approach for the reasons explained below.

Since CoDel performs its queue control operations during the dequeue phase and does not drop incoming packets until the queue is full, it tends to fill its queues more quickly than PIE, which drops packets randomly during the enqueue phase. This is especially true when CoDel has just entered the dropping phase, as it takes time to ramp up its packet dropping frequency. Therefore, the strategy of dropping half of the packets from a fat flow's queue suits FQ-CoDel but is not appropriate for FQ-PIE. Dropping packets in bulk might lead to underutilization of link capacity, as FQ-PIE already enforces queue control during the enqueue phase.

## Dequeue

The packet dequeue process in FQ-PIE is similar to that in FQ-CoDel, where a DRR-based scheduler is used to dequeue packets from each queue. The key difference is that in FQ-CoDel, CoDel operates during this phase, whereas in FQ-PIE, PIE operates during the enqueue phase. The method for obtaining direct measurements of per-packet queue delay is the same in both FQ-PIE and FQ-CoDel, and is performed during the dequeue phase.

## ECN Support

FQ-PIE MAY support ECN by marking ECN-Capable Transport (ECT) packets {{!RFC3168}} instead of dropping them, in accordance with the recommendations in Section 5.1 of {{!RFC8033}}. The Linux, FreeBSD and ns-3 implementations of FQ-PIE comply with these recommendations at the time of writing this document.

# Scope of Experimentation

The design of the FQ-PIE algorithm as described in this document has been a part of the Linux kernel since version 5.6 (released on March 29, 2020), FreeBSD since version 11.0-RELEASE (released on October 10, 2016), and the ns-3 network simulator since version 3.34 (released on July 14, 2021). The following aspects can be explored for further study and experimentation:

- The scenarios similar to those summarized in Figure 4 of {{!RFC7928}} MAY be considered for an in-depth experimentation of FQ-PIE.

- Interactions between flow queuing and new congestion control algorithms, such as Bottleneck Bandwidth and Round-trip propagation time (BBR) {{?I-D.draft-ietf-ccwg-bbr}}.

- Different packet drop probability thresholds to switch from marking packets to dropping packets.

- Evaluation of the enhancements to the PIE algorithm described in Section 5 in {{!RFC8033}} to decide which enhancements are suitable for deployment with FQ-PIE.

- Effectiveness of FQ-PIE in terms of providing isolation and minimal latency for low volume traffic (short flows) such as web applications, instant messaging applications, interactive applications and IoT applications.

- Different hashing mechanisms to improve the overall working of flow queuing.

# Security Considerations

The FQ-PIE algorithm introduces no specific security exposures. The flow queuing aspect of the FQ-PIE algorithm is the same as FQ-CoDel, and hence has similar advantages from the security perspective as outlined in Section 8 of {{!RFC8290}}. The PIE aspect of the FQ-PIE algorithm is the same as described in {{!RFC8033}} that does not have any security exposures.

# IANA Considerations

This document has no IANA actions.

--- back
