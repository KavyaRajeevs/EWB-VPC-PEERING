# AWS VPC Peering: Private Connectivity Between Two VPCs

![AWS](https://img.shields.io/badge/AWS-ap--south--1-orange)
![Architecture](https://img.shields.io/badge/Architecture-VPC%20Peering-blue)
![Instance](https://img.shields.io/badge/AMI-Amazon%20Linux%202023-green)

A hands-on AWS networking project demonstrating **private connectivity
between two isolated VPCs using VPC Peering**, while keeping application
instances private and accessing them securely through a bastion host.

> **Region:** Mumbai (`ap-south-1`)\
> **Student:** Kavya R\
> **Batch:** 1\
> **Completion date:** 18 October 2026\
> **Repository:** https://github.com/KavyaRajeevs/EWB-VPC-PEERING

------------------------------------------------------------------------

## 1. Project Overview

The project creates two VPCs with non-overlapping CIDR ranges:

-   **VPC A:** `10.10.0.0/16`
-   **VPC B:** `10.20.0.0/16`

VPC A contains a public subnet with a bastion host and a private subnet
containing App-A. VPC B contains only a private subnet containing App-B.

The application instances have **no public IPv4 addresses**. Private
communication between them is enabled using a VPC peering connection and
matching routes on both sides.

Access to the private instances is performed through the bastion using
SSH `ProxyJump`.

### Final result

-   App-A and App-B communicate over their private IP addresses.
-   App-B can be reached through the bastion using SSH ProxyJump.
-   Neither App-A nor App-B has a public IPv4 address.
-   Removing App-B's ICMP security-group rule blocks ping, confirming
    that the destination security group controls the traffic.
-   The project resources were cleaned up after validation.

------------------------------------------------------------------------

## 2. Architecture
<img width="6362" height="4957" alt="architecture diagram" src="https://github.com/user-attachments/assets/e66fa3a6-8695-4ed8-8f92-fe6f78abc8f4" />


## 4. Network Design

### VPC A

**CIDR:** `10.10.0.0/16`

  Subnet            CIDR             Type      Purpose
  ----------------- ---------------- --------- --------------
  `ewb-a-public`    `10.10.1.0/24`   Public    Bastion host
  `ewb-a-private`   `10.10.2.0/24`   Private   App-A

### VPC B

**CIDR:** `10.20.0.0/16`

  Subnet            CIDR             Type      Purpose
  ----------------- ---------------- --------- ---------
  `ewb-b-private`   `10.20.1.0/24`   Private   App-B

The CIDRs do not overlap:

-   VPC A → `10.10.0.0/16`
-   VPC B → `10.20.0.0/16`

Non-overlapping CIDRs are required so that a destination such as
`10.20.0.0/16` has an unambiguous routing destination.

------------------------------------------------------------------------

## 5. Internet Gateway and Route Tables

The Internet Gateway `ewb-igw-a` is attached only to **VPC A**.

### Public route table

`ewb-rt-a-public`

  Destination      Target
  ---------------- ------------------
  `10.10.0.0/16`   local
  `0.0.0.0/0`      Internet Gateway
  `10.20.0.0/16`   VPC Peering

The public subnet is considered public because its route table contains
a default route to the Internet Gateway.

### Private route table A

`ewb-rt-a-private`

  Destination      Target
  ---------------- -------------
  `10.10.0.0/16`   local
  `10.20.0.0/16`   VPC Peering

### Private route table B

`ewb-rt-b-private`

  Destination      Target
  ---------------- -------------
  `10.20.0.0/16`   local
  `10.10.0.0/16`   VPC Peering

VPC B has no Internet Gateway. Its private subnet communicates with VPC
A through the peering connection.

------------------------------------------------------------------------

## 6. Security Groups

Security groups restrict traffic to the minimum required sources.

  Security Group     Protocol / Port   Source               Purpose
  ------------------ ----------------- -------------------- ------------------
  `ewb-sg-bastion`   SSH / 22          My public IP `/32`   SSH into bastion
  `ewb-sg-app-a`     SSH / 22          `10.10.1.10/32`      SSH from bastion
  `ewb-sg-app-a`     ICMP echo         `10.20.1.10/32`      Ping from App-B
  `ewb-sg-app-b`     SSH / 22          `10.10.1.10/32`      SSH from bastion
  `ewb-sg-app-b`     ICMP echo         `10.10.2.10/32`      Ping from App-A

The application instances are not exposed directly to the internet.

------------------------------------------------------------------------

## 7. Instances

All instances use **Amazon Linux 2023**.

  Instance       Private IP          Public IP
  ---------- -------------- ------------------
  Bastion      `10.10.1.10`   `43.205.113.255`
  App-A        `10.10.2.10`               None
  App-B        `10.20.1.10`               None

The bastion is the only internet-facing instance.

------------------------------------------------------------------------

## 8. SSH Access Through Bastion

The private instances are accessed using SSH `ProxyJump`.

Example `~/.ssh/config`:

``` sshconfig
Host ewb-bastion
    HostName 43.205.113.255
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    IdentitiesOnly yes

Host ewb-app-a
    HostName 10.10.2.10
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    IdentitiesOnly yes
    ProxyJump ewb-bastion

Host ewb-app-b
    HostName 10.20.1.10
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    IdentitiesOnly yes
    ProxyJump ewb-bastion
```

This allows:

``` bash
ssh ewb-bastion
ssh ewb-app-a
ssh ewb-app-b
```

The private key remains on the laptop and does not need to be copied
onto the bastion.

------------------------------------------------------------------------

## 9. VPC Peering

The peering connection is:

``` text
ewb-a-b-peer
pcx-046305deb7a5add5e
Requester: VPC A
Accepter: VPC B
Status: Active
```

VPC peering provides a private network path between the two VPCs.

### Required routes

Traffic from VPC A to VPC B:

``` text
10.20.0.0/16 → ewb-a-b-peer
```

Traffic from VPC B to VPC A:

``` text
10.10.0.0/16 → ewb-a-b-peer
```

Both sides require routes because network traffic needs a valid forward
and return path.

------------------------------------------------------------------------

## 10. Connectivity Tests

### Test 01 --- Baseline

Before peering and the peer routes were configured:

``` bash
ping -c 4 10.20.1.10
```

Result:

``` text
No replies
100% packet loss
```

This established that the two private networks were not connected before
peering.

### Test 02 --- Forward connectivity

From App-A:

``` bash
ping -c 4 10.20.1.10
```

Result:

``` text
4 replies
0% packet loss
```

### Test 03 --- Reverse connectivity

From App-B:

``` bash
ping -c 4 10.10.2.10
```

Result:

``` text
4 replies
0% packet loss
```

### Test 04 --- Private SSH

From the laptop:

``` bash
ssh ewb-app-b
```

Result:

``` text
Shell accessed through bastion using ProxyJump
```

### Test 05 --- Private instance isolation

EC2 instance details confirmed:

``` text
App-A → Public IPv4: none
App-B → Public IPv4: none
```

### Test 06 --- Controlled security-group failure

The ICMP rule allowing App-A to reach App-B was removed.

Result:

``` text
Ping → FAIL
```

After restoring the exact ICMP rule:

``` text
Ping → PASS
```

This demonstrated that the destination security group was responsible
for allowing or blocking ICMP traffic.

------------------------------------------------------------------------

## 11. Forward and Return Path

### App-A → App-B

The traffic path is:

``` text
App-A
  ↓
App-A outbound security rules
  ↓
ewb-rt-a-private
  ↓
10.20.0.0/16 → VPC Peering
  ↓
ewb-a-b-peer
  ↓
ewb-sg-app-b inbound ICMP rule
  ↓
App-B
```

### App-B → App-A

The return path is:

``` text
App-B
  ↓
Stateful security-group response
  ↓
ewb-rt-b-private
  ↓
10.10.0.0/16 → VPC Peering
  ↓
ewb-a-b-peer
  ↓
App-A
```

### Bastion → App-B

When the laptop executes:

``` bash
ssh ewb-app-b
```

SSH first reaches the bastion at:

``` text
43.205.113.255
```

The bastion then connects to:

``` text
10.20.1.10:22
```

The source seen by App-B is the bastion's private IP:

``` text
10.10.1.10
```

Therefore App-B allows SSH from:

``` text
10.10.1.10/32
```

The public route table also needs the VPC B route:

``` text
10.20.0.0/16 → VPC Peering
```

so the bastion can reach App-B.

------------------------------------------------------------------------

## 12. Troubleshooting

The project intentionally included two failures:

1.  **Baseline failure:** Before peering and routes were configured,
    App-A could not reach App-B.
2.  **Controlled ICMP failure:** Removing App-B's ICMP inbound rule
    blocked the ping.

The controlled failure demonstrates an important troubleshooting
principle:

``` text
Connectivity requires:

Peering
   +
Correct routes
   +
Correct security-group rules
```

Active VPC peering alone does not guarantee connectivity.

------------------------------------------------------------------------

## 13. Cleanup

After validation, the AWS resources were removed.

Cleanup included:

-   Terminating the three EC2 instances.
-   Deleting the VPC peering connection.
-   Deleting the security groups.
-   Deleting both VPCs and associated networking resources.
-   Verifying that no project resources remained.

------------------------------------------------------------------------


## 15. What I Learned

This project demonstrated that VPC peering is only the network
connection between two VPCs. Actual communication requires:

-   Non-overlapping CIDRs.
-   An active peering connection.
-   Correct routes on both sides.
-   Appropriate security-group rules.
-   A valid return path.

I also learned the practical difference between public and private
subnets, particularly how route tables determine whether a subnet has a
path to an Internet Gateway.

Using SSH `ProxyJump` provided secure administrative access to private
instances without giving those instances public IP addresses or copying
the private key onto the bastion.

The controlled ICMP failure also demonstrated how to troubleshoot
connectivity layer by layer instead of assuming that a routing problem
is responsible for every failure.





------------------------------------------------------------------------

## License

This project was created as an AWS networking learning project.
