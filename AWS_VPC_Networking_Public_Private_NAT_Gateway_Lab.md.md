# AWS VPC Networking Hands-On Lab
## Build a Public and Private Network, Deploy a Website, and Test Internet Access Through a NAT Gateway

**Level:** Beginner  
**Platform:** AWS Management Console + EC2 Linux commands  
**Suggested Region:** Frankfurt (`eu-central-1`)  
**Estimated duration:** 2–3 hours  
**Important:** This lab creates billable AWS resources, especially a NAT Gateway. Complete the cleanup section immediately after the exercise.

---

## 1. Lab objective

Build a custom VPC with public and private subnets. Deploy one web server in the public subnet and another EC2 instance in the private subnet. Configure an Internet Gateway, NAT Gateway, route tables, Network ACLs and Security Groups. Then test which systems can be reached from the internet and how a private instance can access the internet through NAT without accepting unsolicited inbound internet connections.

By the end, you will be able to:

- Create a custom VPC and understand its CIDR range.
- Create public and private subnets in separate Availability Zones.
- Attach an Internet Gateway (IGW).
- Configure a NAT Gateway and Elastic IP.
- Configure public and private route tables.
- Configure Security Groups (stateful) and Network ACLs (stateless).
- Launch EC2 instances in public and private subnets.
- Deploy a simple website on the public EC2 instance.
- Connect to the private EC2 instance using the public EC2 instance as a bastion/jump host.
- Test outbound internet connectivity from the private EC2 through the NAT Gateway.
- Verify that the private EC2 cannot be reached directly from the public internet.
- Observe how route tables, Security Groups and NACLs affect network traffic.

---

## 2. Architecture

```text
                         Internet
                            |
                     Internet Gateway
                            |
                 Public Route Table
                 0.0.0.0/0 -> IGW
                            |
          +-----------------+------------------+
          |                                    |
   Public Subnet A                      Public Subnet B
   10.10.1.0/24                         10.10.3.0/24
          |                                    |
   Public EC2 / Web                     NAT Gateway
   10.10.1.x                            Elastic IP
   Public IPv4                                  |
          |                              Private Route Table
          |                              0.0.0.0/0 -> NAT
          |                                    |
          |                              Private Subnet A
          |                              10.10.2.0/24
          |                                    |
          +---- SSH jump connection ---- Private EC2
                                        10.10.2.x
                                        No public IPv4
```

The public EC2 instance hosts the website and can be reached from your browser. It can also be used as a jump host to connect to the private EC2 instance. The private EC2 has no public IPv4 address. It can initiate outbound connections through the NAT Gateway, but the NAT Gateway does not allow an internet host to initiate a new connection to the private EC2.

**Availability Zone note:** Put the NAT Gateway in a public subnet. For this single-NAT teaching lab, route the private subnet to that NAT Gateway. This is suitable for learning, not a highly available production design. Production environments commonly use a NAT Gateway per AZ and keep each private subnet's default route in the same AZ.

---

## 3. IP address plan

Use this address plan consistently throughout the lab.

| AWS resource | CIDR / address | Purpose |
|---|---|---|
| VPC | `10.10.0.0/16` | Entire private network |
| Public Subnet A | `10.10.1.0/24` | Public web/bastion EC2 |
| Private Subnet A | `10.10.2.0/24` | Private application EC2 |
| Public Subnet B | `10.10.3.0/24` | NAT Gateway |
| Public EC2 | Auto-assigned private IP | Website and jump host |
| Private EC2 | Auto-assigned private IP | Private server |
| NAT Gateway | Elastic IP + private IP | Outbound internet for private subnet |

Choose two Availability Zones in the same Region. For example, use `eu-central-1a` and `eu-central-1b` if both are available in your account. AWS account AZ names can map to different physical AZs, so choose from the console rather than assuming a specific mapping.

---

## 4. Create the VPC

1. Sign in to the AWS Management Console.
2. Open **VPC**.
3. Select **Your VPCs → Create VPC**.
4. Choose **VPC only**.
5. Enter:
   - Name tag: `QB-Network-VPC`
   - IPv4 CIDR: `10.10.0.0/16`
   - IPv6 CIDR: No IPv6 CIDR block
   - Tenancy: Default
6. Select **Create VPC**.

### Verify

Open **Your VPCs** and confirm that `QB-Network-VPC` is in the `Available` state.

A VPC is an isolated logical network in an AWS Region. The VPC CIDR defines the private IPv4 address range available for its subnets.

---

## 5. Create the subnets

Open **VPC → Subnets → Create subnet**.

Select `QB-Network-VPC`, then create these three subnets. Select an available AZ for each.

### Public Subnet A

- Name: `QB-Public-Subnet-A`
- Availability Zone: AZ 1
- IPv4 CIDR: `10.10.1.0/24`

### Private Subnet A

- Name: `QB-Private-Subnet-A`
- Availability Zone: AZ 1
- IPv4 CIDR: `10.10.2.0/24`

### Public Subnet B

- Name: `QB-Public-Subnet-B`
- Availability Zone: AZ 2
- IPv4 CIDR: `10.10.3.0/24`

Select each subnet and choose **Actions → Edit subnet settings**. Enable **Auto-assign public IPv4 address** for both public subnets. Leave it disabled for the private subnet.

> A subnet is not public just because its name contains “public” or it has public-IP auto-assignment enabled. It is public when its associated route table has a route to an Internet Gateway. Public IPv4 assignment is a separate setting.

---

## 6. Create and attach an Internet Gateway

1. Open **VPC → Internet Gateways**.
2. Select **Create internet gateway**.
3. Name: `QB-IGW`.
4. Create it.
5. Select `QB-IGW` → **Actions → Attach to a VPC**.
6. Select `QB-Network-VPC` and attach.

An Internet Gateway connects the VPC to the internet. A subnet also needs a route to the IGW, and an EC2 instance needs a public IPv4 address (or Elastic IP) to communicate directly with the public internet over IPv4.

---

## 7. Create the public route table

1. Open **VPC → Route Tables → Create route table**.
2. Name: `QB-Public-RT`.
3. VPC: `QB-Network-VPC`.
4. Create the route table.
5. Select it → **Routes → Edit routes → Add route**.
6. Add:
   - Destination: `0.0.0.0/0`
   - Target: `QB-IGW`
7. Save changes.
8. Open **Subnet associations → Edit subnet associations**.
9. Associate:
   - `QB-Public-Subnet-A`
   - `QB-Public-Subnet-B`
10. Save associations.

The route table should contain:

| Destination | Target |
|---|---|
| `10.10.0.0/16` | local |
| `0.0.0.0/0` | `QB-IGW` |

The `local` route is automatically created and allows communication between subnets in the VPC, subject to Security Group and NACL rules.

---

## 8. Create the NAT Gateway

A NAT Gateway allows instances in a private subnet to initiate connections to the internet, for example to download operating-system updates. It does not provide unsolicited inbound internet access to those private instances.

1. Open **VPC → NAT Gateways → Create NAT Gateway**.
2. Name: `QB-NAT-GW`.
3. Subnet: `QB-Public-Subnet-B`.
4. Connectivity type: **Public**.
5. Choose **Allocate Elastic IP** and allocate a new Elastic IP.
6. Select **Create NAT Gateway**.
7. Wait until its status becomes **Available**.

**Important checks:**

- The NAT Gateway must be in a public subnet.
- The public subnet's route table must have `0.0.0.0/0 → QB-IGW`.
- The NAT Gateway must have an Elastic IP.
- The private route table will point to the NAT Gateway.
- A NAT Gateway has hourly and data-processing charges. Delete it as soon as the lab is complete.

---

## 9. Create the private route table

1. Open **VPC → Route Tables → Create route table**.
2. Name: `QB-Private-RT`.
3. VPC: `QB-Network-VPC`.
4. Create the route table.
5. Select it → **Routes → Edit routes → Add route**.
6. Add:
   - Destination: `0.0.0.0/0`
   - Target: `QB-NAT-GW`
7. Save changes.
8. Open **Subnet associations → Edit subnet associations**.
9. Associate only `QB-Private-Subnet-A`.
10. Save associations.

The private route table should contain:

| Destination | Target |
|---|---|
| `10.10.0.0/16` | `local` |
| `0.0.0.0/0` | `QB-NAT-GW` |

### Compare the route tables

| Route table | Associated subnet(s) | Default route |
|---|---|---|
| `QB-Public-RT` | Public A and Public B | `0.0.0.0/0 → IGW` |
| `QB-Private-RT` | Private A | `0.0.0.0/0 → NAT Gateway` |

This is the key difference between public and private subnet internet routing.

---

## 10. Create Security Groups

Security Groups are stateful virtual firewalls attached to network interfaces/EC2 instances. They allow rules; they do not have explicit deny rules.

### 10.1 Public EC2 Security Group

Create a security group:

- Name: `QB-Public-EC2-SG`
- VPC: `QB-Network-VPC`

**Inbound rules**

| Type | Protocol | Port | Source | Purpose |
|---|---|---:|---|---|
| SSH | TCP | 22 | Your public IP `/32` | Administrator SSH access |
| HTTP | TCP | 80 | `0.0.0.0/0` | Open the website from a browser |

**Outbound rules**

For this lab, retain the default **All traffic → `0.0.0.0/0`** outbound rule.

Do not set SSH source to `0.0.0.0/0`. Use **My IP** in the console. If your public IP changes, update the rule.

### 10.2 Private EC2 Security Group

Create:

- Name: `QB-Private-EC2-SG`
- VPC: `QB-Network-VPC`

**Inbound rules**

| Type | Protocol | Port | Source | Purpose |
|---|---|---:|---|---|
| SSH | TCP | 22 | `QB-Public-EC2-SG` | SSH from the public jump host only |

**Outbound rules**

For this lab, retain the default **All traffic → `0.0.0.0/0`** outbound rule. The private route table and NAT Gateway determine the network path; the Security Group must also allow the outbound traffic.

### Security Group flow

- Browser → public EC2: TCP 80 allowed.
- Your computer → public EC2: TCP 22 allowed only from your IP.
- Public EC2 → private EC2: TCP 22 allowed because the private SG references the public EC2 SG.
- Private EC2 → internet: outbound allowed, then routed through NAT Gateway.
- Internet → private EC2: no public IP and no inbound route through NAT; cannot directly initiate a connection.

---

## 11. Configure Network ACLs

Network ACLs (NACLs) are subnet-level, **stateless** traffic filters. They support allow and deny rules. Because they are stateless, inbound and outbound rules must both permit the traffic, including return traffic.

For a beginner-friendly first run, keep the default NACL associated with the VPC subnets. The default NACL allows all inbound and outbound traffic. First complete the connectivity tests, then create a custom NACL in the optional exercise below.

### Optional: Create a custom NACL for the public and private subnets

If you want to demonstrate NACL behavior:

1. Open **VPC → Network ACLs → Create network ACL**.
2. Name: `QB-Public-NACL`.
3. Select `QB-Network-VPC`.
4. Create it and associate it with `QB-Public-Subnet-A` and `QB-Public-Subnet-B`.
5. Add the following rules, keeping rule numbers unique.

**Public NACL inbound rules**

| Rule # | Type | Protocol | Port range | Source | Allow/Deny |
|---:|---|---|---|---|---|
| 100 | HTTP | TCP | 80 | `0.0.0.0/0` | ALLOW |
| 110 | SSH | TCP | 22 | Your public IP `/32` | ALLOW |
| 120 | Custom TCP | TCP | 1024–65535 | `0.0.0.0/0` | ALLOW |

**Public NACL outbound rules**

| Rule # | Type | Protocol | Port range | Destination | Allow/Deny |
|---:|---|---|---|---|---|
| 100 | Custom TCP | TCP | 1024–65535 | `0.0.0.0/0` | ALLOW |
| 110 | HTTP | TCP | 80 | `0.0.0.0/0` | ALLOW |
| 120 | HTTPS | TCP | 443 | `0.0.0.0/0` | ALLOW |

For a private subnet NACL, create `QB-Private-NACL` and associate it with `QB-Private-Subnet-A`.

**Private NACL inbound rules**

| Rule # | Type | Protocol | Port range | Source | Allow/Deny |
|---:|---|---|---|---|---|
| 100 | SSH | TCP | 22 | `10.10.1.0/24` | ALLOW |
| 110 | Custom TCP | TCP | 1024–65535 | `0.0.0.0/0` | ALLOW |

**Private NACL outbound rules**

| Rule # | Type | Protocol | Port range | Destination | Allow/Deny |
|---:|---|---|---|---|---|
| 100 | HTTP | TCP | 80 | `0.0.0.0/0` | ALLOW |
| 110 | HTTPS | TCP | 443 | `0.0.0.0/0` | ALLOW |
| 120 | Custom TCP | TCP | 1024–65535 | `0.0.0.0/0` | ALLOW |
| 130 | SSH | TCP | `1024–65535` | `10.10.1.0/24` | ALLOW |

**NACL caution:** These are teaching rules, not a complete production allow-list. NACL rule matching is evaluated in ascending rule-number order, and the first match wins. The final `*` rule is an implicit DENY. The ephemeral port range shown here is a common Linux client range; confirm the actual client OS/network requirements before tightening rules. If you apply custom NACLs and connectivity fails, temporarily restore the default allow-all NACL to isolate the issue.

---

## 12. Launch the public EC2 instance

1. Open **EC2 → Instances → Launch instances**.
2. Name: `QB-Public-Web-EC2`.
3. AMI: Amazon Linux 2023.
4. Instance type: choose a small type available to your account, such as `t3.micro` if eligible and quota permits.
5. Key pair: select an existing key pair or create a new one and securely save the private key.
6. Network settings → Edit:
   - VPC: `QB-Network-VPC`
   - Subnet: `QB-Public-Subnet-A`
   - Auto-assign public IP: Enable
   - Firewall: Select existing security group `QB-Public-EC2-SG`
7. Storage: use the smallest practical root volume for the lab.
8. Launch instance.
9. Wait for **Running** and both status checks to pass.
10. Record its public IPv4 address and private IPv4 address.

### Connect to the public EC2

From your terminal:

```bash
chmod 400 your-key.pem
ssh -i your-key.pem ec2-user@PUBLIC_EC2_PUBLIC_IP
```

Replace `your-key.pem` and `PUBLIC_EC2_PUBLIC_IP` with your actual values.

If SSH fails, verify the instance has a public IPv4 address, the public subnet route points to the IGW, the Security Group allows TCP 22 from your current public IP, and the NACL allows the connection and return traffic.

---

## 13. Deploy a basic website on the public EC2

Run these commands on the **public EC2**:

```bash
sudo dnf update -y
sudo dnf install -y httpd
sudo systemctl enable --now httpd
```

Create a simple web page:

```bash
sudo tee /var/www/html/index.html > /dev/null <<'EOF'
<!DOCTYPE html>
<html>
<head>
  <title>AWS VPC Networking Lab</title>
</head>
<body>
  <h1>Public EC2 Website</h1>
  <p>This website is running inside the public subnet.</p>
  <p>Server: QB-Public-Web-EC2</p>
  <p>Network: QB-Network-VPC</p>
</body>
</html>
EOF
```

Check the web service:

```bash
sudo systemctl status httpd
curl -I http://localhost
```

From your laptop browser, open:

```text
http://PUBLIC_EC2_PUBLIC_IP
```

You should see the **Public EC2 Website** page.

If the page does not load, check the EC2 Security Group inbound TCP 80 rule, public route table, IGW attachment, public IPv4 address, NACL rules, and `httpd` service status.

---

## 14. Launch the private EC2 instance

1. Open **EC2 → Instances → Launch instances**.
2. Name: `QB-Private-App-EC2`.
3. AMI: Amazon Linux 2023 (use the same AMI family as the public instance).
4. Instance type: choose a small type supported by your account.
5. Key pair: use the same key pair as the public EC2 for this lab.
6. Network settings → Edit:
   - VPC: `QB-Network-VPC`
   - Subnet: `QB-Private-Subnet-A`
   - Auto-assign public IP: Disable
   - Firewall: Select existing `QB-Private-EC2-SG`
7. Launch the instance.
8. Wait for **Running** and both status checks to pass.
9. Record its private IPv4 address.

**Verify:** The private EC2 should not have a public IPv4 address. Do not assign an Elastic IP to it.

---

## 15. Connect to the private EC2 using the public EC2 as a jump host

You will use the public EC2 as a bastion/jump host. The private key must be available to the SSH client on the public EC2 for the simple `scp` method below. Treat private keys as secrets and remove the copied key after the lab.

### Option A: Copy the key to the public EC2 (simple teaching method)

From your laptop, copy the key to the public EC2:

```bash
scp -i your-key.pem your-key.pem ec2-user@PUBLIC_EC2_PUBLIC_IP:/home/ec2-user/lab-key.pem
```

Then SSH to the public EC2:

```bash
ssh -i your-key.pem ec2-user@PUBLIC_EC2_PUBLIC_IP
```

On the public EC2, set permissions and connect to the private EC2:

```bash
chmod 400 ~/lab-key.pem
ssh -i ~/lab-key.pem ec2-user@PRIVATE_EC2_PRIVATE_IP
```

You should now have a shell on the private EC2.

**Security warning:** Copying a private SSH key onto a bastion is acceptable only as a temporary, controlled lab shortcut. Remove it immediately after the exercise:

```bash
rm -f ~/lab-key.pem
```

For production, prefer AWS Systems Manager Session Manager, EC2 Instance Connect Endpoint, or SSH agent forwarding with carefully managed access rather than storing private keys on a bastion.

### Option B: SSH agent forwarding

If your local SSH client supports an SSH agent, load your key locally and connect using agent forwarding:

```bash
ssh-add your-key.pem
ssh -A -i your-key.pem ec2-user@PUBLIC_EC2_PUBLIC_IP
```

From the public EC2 shell:

```bash
ssh ec2-user@PRIVATE_EC2_PRIVATE_IP
```

Agent forwarding avoids copying the private key file to the bastion, but use it only with a trusted bastion because the remote host can potentially use the forwarded agent while the connection is active.

---

## 16. Test the NAT Gateway concept

This is the main NAT Gateway demonstration. Run the tests from the **private EC2 shell**, not from your laptop or the public EC2.

### Test 1: Confirm the private EC2 has no public IP

In the EC2 console, select `QB-Private-App-EC2` and inspect its networking details.

Expected:
- Private IPv4 address: present (for example, `10.10.2.x`)
- Public IPv4 address: none

### Test 2: Check the private subnet's default route

In **VPC → Route Tables**, select `QB-Private-RT`.

Expected route:

```text
10.10.0.0/16  -> local
0.0.0.0/0     -> nat-xxxxxxxx
```

The NAT Gateway target should be the NAT Gateway created in Step 8.

### Test 3: Check outbound internet connectivity

On the private EC2:

```bash
ip route
curl -I --connect-timeout 5 https://example.com
```

If `curl` is not installed:

```bash
sudo dnf install -y curl
```

Then retry:

```bash
curl -I --connect-timeout 5 https://example.com
```

Expected: an HTTP response such as `HTTP/2 200`, `HTTP/1.1 200`, or a redirect response. The exact response may vary.

### Test 4: Check the public IP seen by an external service

On the private EC2:

```bash
curl -4 https://checkip.amazonaws.com
```

Expected: it should return the NAT Gateway's Elastic IP, not the private EC2's private IP.

Compare the result with the Elastic IP shown under **VPC → NAT Gateways** or **EC2 → Elastic IPs**.

### Test 5: Demonstrate that the private EC2 cannot accept a direct internet connection

- Confirm the private EC2 has no public IPv4 address.
- Do not create a port-forwarding rule on the NAT Gateway; a standard AWS public NAT Gateway does not provide inbound port forwarding to private instances.
- From your laptop, attempting to SSH directly to the private EC2 is not possible because it has no public IP and is not directly reachable from the internet.

The private EC2 can initiate an outbound connection through NAT. Return packets for that connection are allowed back through the NAT state mapping. This is different from an internet host initiating a new connection to the private EC2.

### Test 6: Stop the NAT route temporarily (optional, controlled test)

Only perform this if you understand that it will interrupt outbound internet access from the private subnet.

1. Open `QB-Private-RT` → **Routes → Edit routes**.
2. Delete the `0.0.0.0/0 → NAT Gateway` route and save.
3. From the private EC2, run:

```bash
curl -I --connect-timeout 5 https://example.com
```

Expected: the request should fail or time out because the private subnet no longer has a default route to the internet.

4. Restore the route immediately:

```text
0.0.0.0/0 -> QB-NAT-GW
```

5. Retry the `curl` command. It should work again after the route is restored.

Do not delete the NAT Gateway for this test. Restore the route before proceeding.

---

## 17. Compare public EC2 and private EC2 behavior

Run these commands on each instance.

| Test | Public EC2 | Private EC2 |
|---|---|---|
| Has private IPv4 | Yes | Yes |
| Has public IPv4 | Yes, in this lab | No |
| Default route | IGW | NAT Gateway |
| Website reachable from laptop | Yes, TCP 80 | No direct public access |
| SSH from laptop | Yes, restricted to your IP | No direct internet SSH |
| Outbound internet | Through IGW | Through NAT Gateway |
| Can initiate outbound HTTPS | Yes | Yes, if NAT and rules are configured |
| Can receive unsolicited internet connection | Yes, if SG/NACL/service allow it | No direct public internet path |

**Important:** A public subnet route to an IGW does not automatically expose every EC2 instance. The instance also needs a public IPv4/EIP and permissive enough Security Group and NACL rules. Similarly, a private subnet is not made private by its name; its routing and addressability determine the path.

---

## 18. Demonstrate Security Group vs NACL

### Test A: Security Group blocks website access

1. Open `QB-Public-EC2-SG`.
2. Remove the inbound HTTP TCP 80 rule temporarily.
3. Refresh `http://PUBLIC_EC2_PUBLIC_IP` in your browser.
4. The website should stop responding.
5. Restore the HTTP TCP 80 rule.

### Test B: NACL blocks website access

If you created the custom `QB-Public-NACL`:

1. Open **VPC → Network ACLs → QB-Public-NACL**.
2. Add a DENY rule with a lower rule number than the HTTP allow rule, for example:
   - Rule number: `90`
   - Type: HTTP
   - Protocol: TCP
   - Port: `80`
   - Source: `0.0.0.0/0`
   - Rule action: DENY
3. Refresh the website. It should fail.
4. Delete the temporary deny rule and test again.

Because NACLs are stateless, also remember to allow the return/ephemeral ports in the opposite direction. A rule that allows inbound HTTP but blocks response traffic can still break the website.

### Compare

| Feature | Security Group | Network ACL |
|---|---|---|
| Scope | ENI / EC2 network interface | Subnet |
| Stateful? | Yes | No, stateless |
| Allow rules | Yes | Yes |
| Deny rules | No explicit deny rules | Yes |
| Rule evaluation | All applicable rules are considered | Lowest numbered matching rule wins |
| Return traffic | Automatically allowed for permitted connections | Must be explicitly allowed |

---

## 19. Troubleshooting checklist

### Public website does not open

- Confirm public EC2 is running and status checks pass.
- Confirm it has a public IPv4 address.
- Confirm `QB-Public-RT` is associated with `QB-Public-Subnet-A`.
- Confirm `0.0.0.0/0 → QB-IGW` exists.
- Confirm `QB-IGW` is attached to `QB-Network-VPC`.
- Confirm public EC2 SG allows inbound TCP 80 from your client.
- Confirm NACL allows inbound TCP 80 and outbound ephemeral return traffic.
- Confirm Apache is running:

```bash
sudo systemctl status httpd
sudo ss -lntp | grep ':80'
```

### Cannot SSH from public EC2 to private EC2

- Confirm both EC2 instances are in the same VPC.
- Confirm private EC2 SG inbound SSH rule source is `QB-Public-EC2-SG`.
- Confirm NACL allows TCP 22 and return ephemeral ports.
- Confirm the private EC2 is running and `sshd` is active.
- Use the private EC2's private IPv4 address.

### Private EC2 cannot access the internet

- Confirm NAT Gateway status is **Available**.
- Confirm NAT Gateway is in a public subnet.
- Confirm NAT Gateway has an Elastic IP.
- Confirm public subnet route table has `0.0.0.0/0 → IGW`.
- Confirm private route table has `0.0.0.0/0 → NAT Gateway`.
- Confirm `QB-Private-RT` is associated with the private subnet.
- Confirm private EC2 SG outbound rules allow the test traffic.
- Confirm private NACL outbound allows TCP 80/443 and inbound allows ephemeral return ports.
- Check DNS resolution:

```bash
getent hosts example.com
```

If DNS fails, inspect the VPC's DNS support and DNS hostnames settings, and the instance's resolver configuration. NAT Gateway provides network address translation; it does not itself provide DNS resolution.

### Check the instance's network path

```bash
ip addr
ip route
cat /etc/resolv.conf
```

---

## 20. Cleanup — avoid ongoing AWS charges

**Do not leave the NAT Gateway or other resources running after the lab.** A NAT Gateway can incur hourly and data-processing charges for as long as it exists, even if you are not actively using it.

Recommended cleanup order:

1. **Terminate both EC2 instances**
   - EC2 → Instances
   - Select `QB-Public-Web-EC2` and `QB-Private-App-EC2`
   - Instance state → Terminate instance
2. **Delete the NAT Gateway**
   - VPC → NAT Gateways
   - Select `QB-NAT-GW` → Delete NAT Gateway
   - Wait until deletion completes.
3. **Release the Elastic IP**
   - EC2 → Elastic IPs
   - Disassociate if needed, then release the Elastic IP allocated for the NAT Gateway.
   - Do not release an address still associated with another resource.
4. **Delete custom NACLs** if you created them
   - VPC → Network ACLs
   - Delete `QB-Public-NACL` and `QB-Private-NACL` after disassociating them or re-associating the subnets with the default NACL.
5. **Delete custom route tables**
   - VPC → Route Tables
   - Delete `QB-Public-RT` and `QB-Private-RT` after removing subnet associations.
6. **Detach and delete the Internet Gateway**
   - VPC → Internet Gateways
   - Detach `QB-IGW` from `QB-Network-VPC`, then delete it.
7. **Delete the subnets**
   - VPC → Subnets
   - Delete `QB-Public-Subnet-A`, `QB-Public-Subnet-B`, and `QB-Private-Subnet-A`.
8. **Delete Security Groups**
   - Delete `QB-Public-EC2-SG` and `QB-Private-EC2-SG`.
   - If a group is in use, confirm its ENIs/instances have been removed first.
9. **Delete the VPC**
   - VPC → Your VPCs
   - Delete `QB-Network-VPC`.

### Final billing verification

- Check **EC2 → Instances**: no lab instances remain.
- Check **VPC → NAT Gateways**: no lab NAT Gateway remains.
- Check **EC2 → Elastic IPs**: release unused lab Elastic IPs.
- Check **EC2 → Volumes** and **Snapshots**: delete unattached lab volumes and snapshots only if you no longer need them.
- Check **VPC → Internet Gateways**, **Subnets**, and **Route Tables** for leftover lab resources.
- Check **Billing and Cost Management → Bills** and **Cost Explorer** for any remaining usage.

Terminating an EC2 instance does not always remove separately retained EBS volumes or snapshots. Verify storage resources independently.

---

## 21. Cost planning for this lab

This lab is **not guaranteed to be free**, even if your EC2 instances are eligible for Free Tier benefits. The NAT Gateway is the main cost risk because it has an hourly charge and a data-processing charge. A public IPv4 address, EC2 compute, EBS storage, and data transfer may also be billable.

### Main cost components

| Resource | What can be charged |
|---|---|
| EC2 public instance | Instance running time, EBS storage, public IPv4 address |
| EC2 private instance | Instance running time and EBS storage |
| NAT Gateway | Hourly usage plus data processed |
| Elastic IP / public IPv4 | Public IPv4 hourly usage under current AWS pricing rules |
| S3 | Not required for this lab |
| Data transfer | Internet data transfer, depending on direction and current pricing |
| CloudWatch | Not required for the core lab; extra metrics/logs can add charges |

### Approximate cost for a short demonstration

The exact cost depends on Region, instance types, hours, NAT data processed, public IPv4 charges, account credits and current AWS pricing. For planning only, a small two-instance lab in `eu-central-1` with a NAT Gateway left running for a full 24 hours can cost **several US dollars per day**, and the NAT Gateway can be a substantial part of the total. A short 1–2 hour exercise is much cheaper than leaving it running all day, but do not treat this as a price quote.

Before launching:

1. Open the AWS Pricing Calculator.
2. Select the Frankfurt Region (`eu-central-1`).
3. Add two EC2 instances with the planned instance type and hours.
4. Add EBS storage for both instances.
5. Add one NAT Gateway and estimate its running hours and data processed.
6. Include public IPv4 addresses and any expected data transfer.
7. Compare the estimate with your account's Free Tier/credits page.

**Cost-saving recommendations**

- Create the NAT Gateway only when you are ready to test it.
- Keep the exercise short and delete the NAT Gateway immediately after testing.
- Use small EC2 instance types that your account quota supports.
- Do not add a Load Balancer, NAT Gateway per AZ, or optional monitoring services to this beginner lab.
- Create an AWS Budget alert before starting. A budget alert is a notification, not a hard spending cap.
- Complete the cleanup checklist and verify deletion in the console.

---

## 22. Student completion checklist

- [ ] Created `QB-Network-VPC` with CIDR `10.10.0.0/16`.
- [ ] Created two public subnets and one private subnet.
- [ ] Attached `QB-IGW` to the VPC.
- [ ] Configured `QB-Public-RT` with a default route to the IGW.
- [ ] Created `QB-NAT-GW` in a public subnet and allocated an Elastic IP.
- [ ] Configured `QB-Private-RT` with a default route to the NAT Gateway.
- [ ] Created and tested public and private EC2 Security Groups.
- [ ] Understood the default NACL and optionally tested custom NACL rules.
- [ ] Launched the public EC2 and deployed the website.
- [ ] Opened the website from a laptop browser.
- [ ] Launched the private EC2 without a public IPv4 address.
- [ ] Connected to the private EC2 through the public jump host.
- [ ] Verified outbound internet access from the private EC2 through NAT.
- [ ] Verified the public IP returned by the private EC2 is the NAT Gateway Elastic IP.
- [ ] Temporarily removed and restored the NAT route to observe the difference.
- [ ] Cleaned up all billable resources.

---

## 23. Key concepts to remember

- **VPC:** Your isolated virtual network in AWS.
- **Subnet:** A range of IP addresses inside a VPC, placed in one Availability Zone.
- **Public subnet:** A subnet whose route table has a route to an Internet Gateway. For direct IPv4 internet access, the instance also needs a public IPv4/EIP and suitable security rules.
- **Private subnet:** A subnet without a direct route to an Internet Gateway. It may use a NAT Gateway for outbound internet access.
- **Internet Gateway:** Enables internet connectivity for resources with public addressing and suitable routes/rules.
- **NAT Gateway:** Allows private-subnet resources to initiate outbound internet connections without exposing their private IP addresses to direct inbound internet connections.
- **Route Table:** Decides where network traffic is sent.
- **Security Group:** Stateful firewall attached to an EC2 network interface.
- **Network ACL:** Stateless subnet-level firewall with allow and deny rules.

### Official AWS documentation

- VPC: https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html
- Subnets: https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html
- Internet Gateway: https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html
- NAT Gateway: https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html
- Route tables: https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html
- Security Groups: https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html
- Network ACLs: https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html
- AWS Pricing Calculator: https://calculator.aws/
- AWS VPC pricing: https://aws.amazon.com/vpc/pricing/
