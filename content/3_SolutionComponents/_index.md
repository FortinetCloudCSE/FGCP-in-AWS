---
title: "Solution Components"
chapter: false
menuTitle: "Solution Components"
weight: 30
---

FGCP HA provides AWS networks with enhanced reliability through device fail-over protection, link fail-over protection, and remote link fail-over protection. In addition, reliability is further enhanced with session fail-over protection for most IPv4 and IPv6 sessions including TCP, UDP, ICMP, IPsec\SSL VPN, and NAT sessions.

A FortiGate FGCP cluster appears as a single logical FortiGate instance and configuration synchronization allows you to configure a cluster in the same way as a standalone FortiGate unit. If a fail-over occurs, the cluster recovers quickly and automatically and can also send notifications to administrator so that the problem that caused the failure can be corrected and any failed resources restored.

The FortiGate instances will use multiple interfaces for data plane and control plane traffic to achieve FGCP clustering in an AWS VPC.  The FortiGate instances require four ENIs for this solution to work as designed so make sure to use an AWS EC2 instance type that supports this.   Reference AWS Documentation for further information on this.

For data plane functions the FortiGates will use two dedicated ENIs, one for a public interface (ie ENI0\port1) and another for a private interface (ie ENI1\port2).  These ENIs will utilize primary IP addressing and FortiOS should not sync the interface configuration (config system interface) or static routes (config router static) as these FGTs are in separate subnets.  This is controlled via the CLI (config system vdom-exception). Thus when configuring these items, you should do so individually on both FortiGates.

A cluster EIP will be associated to the primary IP of the public interface (ie ENI0\port1) of the current master FortiGate instance and will be reassociated to a new master FortiGate instance as well.

For control plane functions, the FortiGates will use a dedicated ENI (ie ENI2\port3) for HA management access to each instance and also allow each instance to independently and directly communicate with the public AWS EC2 API.  This dedicated interface is critical to failing over AWS SDN properly when a new FGCP HA master is elected and is the only method of access available to the current slave FortiGate instance.

Depending on the FortiOS version, the FortiGates will either reuse the HA management interface (ie ENI2\port3) or use another dedicated ENI (ie ENI3\port4) for FGCP HA communication to perform tasks such as heartbeat checks, configurationport4 sync, and session sync.  In FortiOS 7.0.x and newer versions, one ENI will be used for both HA management and communication channels.

The FortiGates are configured to use the unicast version of FGCP by applying the configuration below on both the master and slave FortiGate instances.  This configuration is automatically configured and bootstrapped to the instances when deployed by the provided CloudFormation or Terraform Templates.

#### Example Master FGCP Configuration:
    config system vdom-exception
    edit 1
    set object sytem.interface
    next
    edit 2
    set object router.static
    next
    end
    config system ha
    set group-name "group1"
    set mode a-p
    set hbdev "port3" 50
    set session-pickup enable
    set ha-mgmt-status enable
    config ha-mgmt-interface
    edit 1
    set interface port3
    set gateway 10.0.3.1
    next
    end
    set override disable
    set priority 255
    set unicast-hb enable
    set unicast-hb-peerip 10.0.30.10
    end

#### Example Slave FGCP Configuration:
    config system vdom-exception
    edit 1
    set object sytem.interface
    next
    edit 2
    set object router.static
    next
    end
    config system ha
    set group-name "group1"
    set mode a-p
    set hbdev "port3" 50
    set session-pickup enable
    set ha-mgmt-status enable
    config ha-mgmt-interface
    edit 1
    set interface port3
    set gateway 10.0.30.1
    next
    end
    set override disable
    set priority 1
    set unicast-hb enable
    set unicast-hb-peerip 10.0.3.10
    end

{{% notice note %}}
Note the **config system vdom-exception** section.  Since the FortiGates are deployed in different Availability Zones, they are in separate subnets and can't share or sync their physical interface IPs or static routes due to next hop IP address being different.  Reference [vdom-exceptions](https://docs.fortinet.com/document/fortigate/7.4.0/administration-guide/105611/vdom-exceptions) to find out more.  In general, you will only need this for physical interfaces and static routes.  Enabling this for other items such as VIPs can add complexity when working with [FortiManager](https://community.fortinet.com/t5/FortiManager/Technical-Tip-FortiManager-manages-FortiGate-HA-in-AWS-Azure/ta-p/260388).
{{% /notice %}}

The FortiGate instances will make calls to the public AWS EC2 API to update AWS SDN to failover both inbound and outbound traffic flows to the new master FortiGate instance.  There are a few components that make this possible.

FortiOS will assume IAM permissions to access the AWS EC2 API by using the IAM instance role attached to the FortiGate instances.  The instance role is what grants the required permissions for FortiOS to:
  - Reassign cluster EIPs assigned to primary IPs assigned to the data plane ENIs
  - Update existing routes to target the new master instance ENIs

The FortiGate instances will utilize their independent and direct internet access available through the FGCP HA management interface (ie ENI2\port3) to access the public AWS EC2 API.  It is critical that this ENI is in a public subnet with an EIP assigned so that each instance has independent and direct access to the internet or the AWS SDN will not reference the current master FortiGate instance which will break data plane traffic.

{{% notice note %}}
There is an option to deploy the FortiGates to where the FGCP HA management interface (ie ENI2\port3) can access AWS EC2 API via private VPC endpoints and would not require dedicated EIPs.  However, this comes with caveats to consider.

First, a dedicated method of access to the FortiGate instances needs to be setup to allow dedicated access to the HAmgmt interfaces.  This method of access should not use the master FortiGate instance so that either instance can be accessed regardless of the cluster status.  Examples of dedicated access are [Direct Connect](https://aws.amazon.com/directconnect/), IPsec VPN connections to an attached [AWS VPN Gateway](https://docs.aws.amazon.com/vpc/latest/userguide/SetUpVPNConnections.html), or using [Transit Gateway](https://aws.amazon.com/transit-gateway/).  Reference AWS Documentation for further information.

Second, the FortiGates should be configured to use the ‘169.254.169.253’ IP address for the AWS intrinsic DNS server as the primary DNS server to allow proper resolution of AWS API hostnames during failover to a new master FortiGate.  Here is an example of how to configure this with CLI commands:
```
config system dns
set primary 169.254.169.253
end
```

Finally, the VPC interface endpoint needs to be deployed into both of the HAmgmt subnets and must also have ‘Private DNS’ enabled to allow DNS resolution of the default AWS EC2 API public hostname to the private IP address of the VPC endpoint.  This means that the VPC also needs to have both DNS resolution and hostname options enabled as well.  Reference [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/vpce-interface.html#vpce-private-dns) for further information.
{{% /notice %}}

For further details on FGCP and its components, reference the High Availability chapter in the FortiOS Handbook on the [Fortinet Documentation site](https://docs.fortinet.com).
