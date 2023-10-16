---
title: "Overview"
menuTitle: "Overview"
weight: 20
---

FortiOS supports using FGCP (FortiGate Clustering Protocol) in unicast form to provide an active-passive clustering solution for deployments in AWS.  This feature shares a majority of the functionality that FGCP on FortiGate hardware provides with key changes to support AWS SDN (Software Defined Networking).

This solution works with two FortiGate instances configured as a master and slave pair and that the instances are deployed in different subnets and different availability zones within a single VPC.  These FortiGate instances act as a single logical instance and do not share interface IP addressing as they are in different subnets.  You can deploy these in the same subnets and availability zone if desired and share secondary interface IP addressing, however this is a less common design.

The main benefits of this solution are:
  - Fast and stateful failover of FortiOS and AWS SDN without external automation\services
  - Automatic AWS SDN updates to EIPs and route targets (and ENI secondary IPs, for single AZ)
  - Native FortiOS session sync of firewall, IPsec\SSL VPN, and VOIP sessions
  - Native FortiOS configuration sync
  - Ease of use as the cluster is treated as single logical FortiGate

For further information on FGCP reference the High Availability chapter in the FortiOS Handbook on the [Fortinet Documentation site](https://docs.fortinet.com).

**Note:**  Other Fortinet solutions for AWS such as FGCP HA (Single AZ), AutoScaling, and Transit Gateway are available.  Please visit [www.fortinet.com/aws](https://www.fortinet.com/aws) for further information.
