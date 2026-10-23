# aws-vpc-peering-project

AWS VPC Peering — Private Connectivity Lab
Project: EWB VPC Peering
Course: EWB Courses — AWS Architect Associate, Project 01
Region: Asia Pacific (Sydney) — ap-southeast-2
Completed: 23 October 2026

This project demonstrates how to connect two Amazon Virtual Private Clouds (VPCs) using VPC Peering, while keeping application instances on private IP addresses. It also verifies the forward and return paths, private SSH access through a bastion host, and the effect of route-table and security-group changes.
Objectives
- Create two VPCs with non-overlapping CIDR ranges.
- Connect them using a VPC peering connection.
- Configure routes in both VPCs.
- Keep application EC2 instances private, without public IPv4 addresses.
- Access private instances through a bastion host using SSH ProxyJump.
- Test connectivity and demonstrate controlled failures and recovery.

                           My laptop
                            |
                         SSH / 22
                            |
                    +----------------+
                    |  Bastion host  |
                    |  VPC A public  |
                    |  10.10.1.10    |
                    +----------------+
                            |
              +-------------+------------------+
              |                                |
       +------v-------+                 VPC Peering
       |   VPC A      |===============================+
       | 10.10.0.0/16 |                               |
       +--------------+                        +------v-------+
       | Public subnet|                        |    VPC B     |
       |10.10.1.0/24  |                        | 10.20.0.0/16 |
       +--------------+                        +--------------+
       | Private      |                        | Private      |
       | subnet       |                        | subnet       |
       |10.10.2.0/24  |                        |10.20.1.0/24  |
       +------+-------+                        +------+-------+
              |                                       |
       +------v-------+                        +------v-------+
       |   App-A      |<--- private IP traffic ->|    App-B     |
       | 10.10.2.10   |                        | 10.20.1.10   |
       | No public IP |                        | No public IP |
       +--------------+                        +--------------+

       Internet Gateway is attached to VPC A only.
       VPC B has no Internet Gateway.

  AWS Resources
Resource	Configuration
VPC A	ewb-vpc-a — 10.10.0.0/16
VPC B	ewb-vpc-b — 10.20.0.0/16
VPC A public subnet	10.10.1.0/24
VPC A private subnet	10.10.2.0/24
VPC B private subnet	10.20.1.0/24
Bastion host	10.10.1.10 — public access restricted by SSH security-group rule
App-A	10.10.2.10 — no public IPv4
App-B	10.20.1.10 — no public IPv4
Peering	One active VPC peering connection between VPC A and VPC B


For security, this README intentionally omits account-specific AWS resource IDs and public IP addresses. Keep private keys out of the repository.
Route Table Configuration
Route table	Destination	Target	Purpose
VPC A public	10.10.0.0/16	local	Local VPC A traffic
VPC A public	0.0.0.0/0	Internet Gateway	Bastion internet connectivity
VPC A public	10.20.0.0/16	VPC peering connection	Bastion-to-VPC B traffic
VPC A private	10.10.0.0/16	local	Local VPC A traffic
VPC A private	10.20.0.0/16	VPC peering connection	App-A to VPC B
VPC B private	10.20.0.0/16	local	Local VPC B traffic
VPC B private	10.10.0.0/16	VPC peering connection	Return path to VPC A


Key lesson: Both sides need the correct routes. An active peering connection alone does not guarantee connectivity.
Security Group Rules
The project used restrictive inbound rules for SSH and ICMP testing.
Security group	Inbound traffic	Source
Bastion	SSH / TCP 22	Trusted client public IP as /32
App-A	SSH / TCP 22	Bastion private IP 10.10.1.10/32
App-A	ICMP Echo Request	App-B private IP 10.20.1.10/32
App-B	SSH / TCP 22	Bastion private IP 10.10.1.10/32
App-B	ICMP Echo Request	App-A private IP 10.10.2.10/32


Security note: Do not expose SSH to 0.0.0.0/0. Use a trusted client IP /32, and do not commit SSH private keys or sensitive AWS details to GitHub.

Private SSH Through a Bastion
SSH ProxyJump allows the laptop to connect to private instances through the bastion. The private key stays on the laptop.

Example ~/.ssh/config (replace the example values with your own local configuration):

Host ewb-bastion
    HostName <BASTION_PUBLIC_IP>
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem

Host ewb-app-a
    HostName 10.10.2.10
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    ProxyJump ewb-bastion

Host ewb-app-b
    HostName 10.20.1.10
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    ProxyJump ewb-bastion


    ssh ewb-bastion
ssh ewb-app-a
ssh ewb-app-b

The SSH configuration above is a template; it assumes the instances use the ec2-user account and that the local key path is correct.
Connectivity Testing and Results

| Test | Action | Observed result |
|---|---|---|
| Baseline route failure | Remove the VPC B route from App-A's route table, then ping App-B | 100% packet loss |
| Forward path | Restore the route and ping App-B from App-A | 4 packets received, 0% loss |
| Reverse path | Ping App-A from App-B | 4 packets received, 0% loss |
| Private SSH | Connect to App-B using `ProxyJump` | Shell reached through the bastion |
| Private-instance isolation | Inspect App-A and App-B instance details | Neither instance has a public IPv4 address |
| Security-group failure | Remove App-B's ICMP Echo Request rule, ping, then restore the rule | Ping fails while the rule is absent and succeeds after restoration |

Example test commands:

# Run on App-A
ping -c 4 10.20.1.10

# Run on App-B
ping -c 4 10.10.2.10

# From the laptop, using the SSH config above
ssh ewb-app-b


What I Learned
1. VPC peering requires non-overlapping CIDR ranges.
2. Route tables need routes to the peer VPC on both sides.
3. Security groups must allow the required traffic from the correct private IPs.
4. A working forward route without a return route is not enough.
5. A bastion host and SSH ProxyJump provide a way to reach private EC2 instances without assigning them public IP addresses.
6. VPC peering is non-transitive and does not let VPC B use VPC A's Internet Gateway as its own internet route.
7. Removing and restoring one route or security-group rule is a useful way to isolate a connectivity issue.
Technologies Used
- Amazon Web Services (AWS)
- Amazon VPC
- VPC Peering
- EC2
- Public and private subnets
- Route tables
- Internet Gateway
- Security Groups
- Linux ping
- SSH and ProxyJump
Repository Evidence
Add sanitized screenshots from your project report to a folder such as screenshots/ and link them here. Suggested evidence:
- VPC peering connection in Active state
- Route tables and subnet associations
- Security-group inbound rules
- Successful forward and reverse ping tests
- SSH connection through the bastion
- Failed connectivity test before restoring a route or ICMP rule
Example:

![Active VPC Peering](screenshots/vpc-peering-active.png)
![Route Table](screenshots/route-table.png)
![Connectivity Test](screenshots/ping-success.png)

Only add screenshots after placing the image files in the repository. Hide account IDs, public IPs, and any other sensitive information before publishing.
Author: Arjit Kumar Pandey
Project: EWB VPC Peering — AWS Architect Associate
