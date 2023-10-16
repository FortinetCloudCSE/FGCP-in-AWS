---
title: "FAQs"
menuTitle: "FAQs"
weight: 80
---


  - **Does FGCP support having multiple Cluster EIPs and secondary IPs on ENI0\port1?**

Yes.  FGCP will move over any secondary IPs associated to ENI0\port1 and EIPs associated to those secondary IPs to the new master FortiGate instance.  You will need to configure secondary IPs on the ENI via the AWS EC2 Console and in FortiOS for port1.  The private IPs configured on the ENI and FortiOS must match.

  - **Does FGCP support having multiple routes for ENI1\port2?**

Yes.  FGCP will move any routes (regardless of the network CIDR) found in AWS route tables that are referencing any of the current master FortiGate instance’s data plane ENIs (ENI0\port1, ENI1\port2, etc) to the new master on a failover event.

  - **What VPC configuration is required when deploying either of the existing VPC CloudFormation templates?**

The existing VPC CloudFormation template is expecting the same VPC configuration that is provisioned in the new Base VPC template.  The existing customer VPC would need to have 4 subnets in each of the two availability zones to cover the required Public, Private, HAsync, and HAmgmt subnets.  Also ensure that an S3 gateway endpoint deployed and assigned to both of the PublicSubnet's AWS route table.  Another critical point is that all of the Public and HAmgmt subnets need to be configured as public subnets.  This means that an IGW needs to be attached to the VPC and a route table, with a default route using the IGW, needs to be associated to the Public and HAmgmt subnets.

  - **During a failover test we see successful failover to a new master FortiGate instance, but then when the original master is online, it becomes master again.**

The master selection process of FGCP will ignore HA uptime differences unless they are larger than 5 minutes.  The HA uptime is visible in the GUI under System > HA.  This is expected and the default behavior of FortiOS but can be changed in the CLI under the ‘config system ha’ table.  For further details on FGCP master selection and how to influence the process, reference primary unit selection section of the High Availability chapter in the FortiOS Handbook on the [Fortinet Documentation site](https://docs.fortinet.com/).

  - **During a failover test we see FGCP select a new master but AWS SDN is not updated to point to the new master FortiGate instance.**

Confirm the FortiGates configuration are in-sync and are properly selecting a new master by seeing the HA role change as expected in the GUI under System > HA or CLI with ‘get sys ha status’.  However during a failover the routes, and Cluster EIPs are not updated, then your issue is likely to do with direct internet access via HAmgmt interface (ENI2\port3) of the FortiGates, failed DNS resolution, or IAM instance role permissions issues.

For details on the IAM instance profile configuration that should be used, reference the policy statement attached to the ‘iam-role-policy’ resource in any of the CloudFormation templates.

For the HAmgmt interface, confirm this is configured properly in FortiOS under the ‘config system ha’ section of the CLI.  Reference the example master\slave CLI HA configuration in the Solutions Components section of this document.

Also confirm that subnet the HAmgmt interface is associated to, is a subnet with public internet access and that this interface has an EIP associated to it.  This means that an IGW needs to be attached to the VPC, and a route table with a default route to the IGW needs to be associated to the HAmgmt subnet.

Finally, the AWS API calls can be debugged on the FortiGate instance that is becoming master with the following CLI commands:
```
diag deb app awsd -1
diag deb enable
```

This can be disabled with the following CLI commands:
```
diag deb app awsd 0
diag deb disable
```

  - **Is it possible to validate that both FortiGate instances are able to assume the IAM instance role and successfully and access AWS EC2 API endpoints without a failover event?.**

Yes.  You can run the following command on either FortiGate to confirm availability.  Remember that all DNS queries and HTTPS calls to AWS EC2 API endpoints will be sent out of the HAmgmt interfaces.  To get more verbose output, you can use the diag debug commands shown above.

```
diag test app awsd 4
```

  - **Is it possible to remove direct internet access from the HAmgmt subnet and provide private AWS EC2 API access via a VPC interface endpoint?**

Yes.  However, there are a few caveats to consider.

First, a dedicated method of access to the FortiGate instances needs to be setup to allow dedicated access to the HAmgmt interfaces.  This method of access should not use the master FortiGate instance so that either instance can be accessed regardless of the cluster status.  Examples of dedicated access are Direct Connect or IPsec VPN connections to an attached AWS VPN Gateway.  Reference [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/SetUpVPNConnections.html) for further information.

Second, the FortiGates should be configured to use the ‘169.254.169.253’ IP address for the AWS intrinsic DNS server as the primary DNS server to allow proper resolution of AWS API hostnames during failover to a new master FortiGate.  Here is an example of how to configure this with CLI commands:
```
config system dns
set primary 169.254.169.253
end
```

Finally, the VPC interface endpoint needs to be deployed into both of the HAmgmt subnets and must also have ‘Private DNS’ enabled to allow DNS resolution of the default AWS EC2 API public hostname to the private IP address of the VPC endpoint.  This means that the VPC also needs to have both DNS resolution and hostname options enabled as well.  Reference [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/vpce-interface.html#vpce-private-dns) for further information.

  - **Is it possible to further restrict the IAM policy used by the FortiGates to specific resources?**

Yes.  You can use resource definitions like shown below to restrict the actions of 'ec2:ReplaceRoute' and 'ec2:AssociateAddress' to specific VPC route table IDs and a set of interfaces, EIP allocation IDs, and instances.  Reference [AWS Documentation](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonec2.html) for further information. 

```
{
	"Version": "2012-10-17",
	"Statement": [{
			"Sid": "BootStrapFromS3",
			"Effect": "Allow",
			"Action": [
				"s3:GetObject"
			],
			"Resource": "*"
		},
		{
			"Sid": "SDNConnectorFortiView",
			"Effect": "Allow",
			"Action": [
				"ec2:DescribeRegions",
				"eks:DescribeCluster",
				"eks:ListClusters",
				"inspector:DescribeFindings",
				"inspector:ListFindings"
			],
			"Resource": "*"
		},
		{
			"Sid": "HAGatherInfo",
			"Effect": "Allow",
			"Action": [
				"ec2:DescribeAddresses"
				"ec2:DescribeInstances",
				"ec2:DescribeRouteTables",
				"ec2:DescribeVpcEndpoints"
			],
			"Resource": "*"
		},
		{
			"Sid": "FailoverEIPs",
			"Effect": "Allow",
			"Action": "ec2:AssociateAddress",
			"Resource": [
				"arn:aws:ec2:us-east-2:123456789012:elastic-ip/eipalloc-0dfae290176fc15d9",
				"arn:aws:ec2:us-east-2:123456789012:network-interface/*",
				"arn:aws:ec2:us-east-2:123456789012:instance/*"
			]
		},
		{
			"Sid": "FailoverVPCroutes",
			"Effect": "Allow",
			"Action": "ec2:ReplaceRoute",
			"Resource": "arn:aws:ec2:us-east-2:123456789012:route-table/rtb-0d5d1757917c71c6e"
		}
	]
}
```

  - **Is it possible to restrict the instance metadata service (IMDS) version to v2 only?**

Yes.  Starting in FortiOS 6.4.3 GA, you can modify this via the AWS CLI command 'aws ec2 modify-instance-metadata-options' to set '--http tokens' to required.  This disables the v1 version of the IMDS for that instance.  Reference [AWS Documentation](https://docs.aws.amazon.com/cli/latest/reference/ec2/modify-instance-metadata-options.html) for further information.

```
aws ec2 modify-instance-metadata-options --instance-id i-01234567890123456 --http-tokens required --region us-west-2