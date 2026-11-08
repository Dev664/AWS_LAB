# AWS Hands-On Lab: S3 + EC2 Website + Auto Scaling + CloudWatch + SNS

**Level:** Beginner-friendly \| **Suggested Region:** Frankfurt
(`eu-central-1`) \| **Estimated time:** 3--5 hours

## 1. Project overview

In this lab, you will store website data in an Amazon S3 bucket. An EC2
instance will run a small Python website and retrieve the data from S3.
An Application Load Balancer (ALB) will distribute requests across EC2
instances. An Auto Scaling Group (ASG) will add or remove instances as
CPU demand changes. CloudWatch will monitor CPU and application logs,
and an SNS email subscription will receive an alert when the ASG's
average CPU is above 20%.

You do not need a programming background. Follow the console
instructions and copy/paste the supplied files and commands.

### What you will learn

-   Create an S3 bucket and upload JSON data.
-   Give EC2 secure S3 access using an IAM role (no hard-coded AWS
    keys).
-   Build a Python/Flask website.
-   Create an AMI, target group, ALB, Launch Template and Auto Scaling
    Group.
-   Generate CPU load and observe scale-out.
-   Monitor EC2 CPU and application logs in CloudWatch.
-   Create a CPU alarm and send an email through SNS.
-   Delete resources safely after the lab.

## 2. Architecture

``` text
                       WEBSITE USERS
                            |
                            v
                 APPLICATION LOAD BALANCER
                            |
                            v
                    AUTO SCALING GROUP
                    Min 1 | Desired 2 | Max 4
                       /     |     \
                      v      v      v
                    EC2-1  EC2-2  EC2-3
                   Flask  Flask  Flask
                      \      |      /
                       \     |     /
                         IAM ROLE
                            |
                            v
                         S3 BUCKET
                       products.json

 EC2 CPU metrics ----------------------> CloudWatch
 Application log file -----------------> CloudWatch Logs
 ASG average CPU > 20% ----------------> CloudWatch Alarm
                                              |
                                              v
                                             SNS
                                              |
                                              v
                                         EMAIL INBOX
```

**Request flow:** A visitor opens the ALB address. The ALB forwards the
request to a healthy EC2 instance. Flask asks S3 for `products.json`,
then displays the returned data. CloudWatch monitors the servers and
application logs. Auto Scaling adjusts capacity, and SNS sends alarm
emails.

## 3. AWS resources

  Service                     Purpose
  --------------------------- ----------------------------------------
  S3                          Stores product data
  EC2                         Runs the website
  IAM                         Controls EC2 permissions
  AMI                         Reusable copy of the configured server
  Launch Template             Instructions for launching instances
  Auto Scaling Group          Adds/removes EC2 capacity
  Application Load Balancer   Distributes web requests
  CloudWatch                  Collects metrics and logs
  CloudWatch Alarm            Detects CPU above 20%
  SNS                         Sends email notifications

## 4. Before starting

You need an AWS account with permission to create these resources, a web
browser, an email inbox you can access, and an SSH client if using SSH
(Windows Terminal, PuTTY or MobaXterm).

### Cost warning

This lab can incur charges for EC2, EBS volumes and snapshots, the
Application Load Balancer, public IPv4 addresses, S3, CloudWatch
metrics/logs, and optional NAT Gateway/VPC endpoints. A load balancer
can cost money while it exists, even with little traffic. Detailed
monitoring and log ingestion/storage may add charges. Stopping EC2 does
not remove EBS or all other charges.

Use small instance types such as `t3.micro` if suitable and available.
Set an AWS Budget alert before starting if possible (a budget alert is
not a spending cap). Do not use production data. **Complete Part J
cleanup immediately after the exercise.**

### Estimated cost for one day (24 hours)

This lab is possible with small instances, but **do not assume the complete setup will be free**. Free Tier eligibility depends on account creation date, current credits/benefits, Region and usage. Check **Billing and Cost Management → Free Tier / Credits** before launching resources.

| Cost item | What causes the charge |
|---|---|
| EC2 | Original web server plus Auto Scaling Group instances. Keeping the original instance running means a baseline of 3 instances (1 original + 2 ASG instances). |
| EBS | Root disks attached to EC2; storage can remain billable after an instance is stopped. |
| Application Load Balancer | Charged while it exists, based on hours and load balancer capacity units (LCUs). |
| Public IPv4 | Public IPv4 addresses can be charged hourly. |
| S3 | Storage and requests; normally small for this demo. |
| CloudWatch | Basic EC2 metrics are generally available without detailed monitoring; log ingestion/storage, custom metrics and alarms can add charges. |
| SNS | Email notifications are usually low-cost at this scale; regional pricing and usage apply. |
| CPU load test | T3 Unlimited may incur surplus CPU-credit charges if the test creates surplus credit usage. |

**Indicative 24-hour planning estimate: US$2.10–US$3.30 (approximately ₹190–₹290), before taxes.** This is not a quote or guarantee. It assumes small `t3.micro`-class instances, three EC2 instances for most of the day, small EBS volumes, one ALB for 24 hours with low traffic, public IPv4 usage, small S3 data, limited CloudWatch logs, and no NAT Gateway. Scaling out while the original EC2 remains running can increase the cost. Exact charges depend on Frankfurt pricing, instance hours, public IPs, ALB LCUs, log volume, exchange rates and your account's credits.

To reduce cost:
1. Once the AMI is **Available** and the Launch Template is verified, consider terminating the original EC2 instance if it is no longer needed.
2. For a short demonstration, use ASG minimum `1`, desired `1` or `2`, and maximum `2` or `3`, subject to your vCPU quota.
3. Keep the ALB only for the demonstration and delete it immediately afterwards.
4. Generate CPU load for only a few minutes and stop the process.
5. Use basic monitoring and keep log volume/retention low.
6. Avoid a NAT Gateway for this introductory lab; consider an S3 Gateway VPC Endpoint where appropriate.
7. Create an AWS Budget alert. It sends notifications; it is **not** a hard spending cap.
8. Complete the cleanup section and verify all resources are deleted.

For an exact estimate, enter the Region, instance types/hours, EBS sizes, ALB hours/LCUs, public IPv4 hours and CloudWatch usage in the AWS Pricing Calculator. Check current prices and your account's Free Tier/credit status before starting.

Use one Region for all resources. This guide uses `eu-central-1`
(Frankfurt). Check the Region selector in the AWS Console before
creating resources.

### Resource names

  Resource             Name
  -------------------- -------------------------------
  S3 bucket            `qb-web-data-<unique-number>`
  IAM policy           `QB-EC2-S3-Read-Policy`
  IAM role             `QB-EC2-S3-Role`
  Original EC2         `QB-S3-WebServer`
  AMI                  `QB-S3-WebServer-AMI`
  ALB security group   `QB-ALB-SG`
  Target group         `QB-Web-TG`
  Load balancer        `QB-Web-ALB`
  Launch Template      `QB-S3-Web-LT`
  Auto Scaling Group   `QB-S3-Web-ASG`
  SNS topic            `QB-EC2-CPU-Alert`
  CloudWatch alarm     `QB-ASG-CPU-Above-20`
  Log group            `/qb/s3-web-app`

Replace `<unique-number>` with a number or random suffix. S3 bucket
names must be globally unique. Whenever this guide says
`YOUR-BUCKET-NAME`, use the exact bucket name you created.

------------------------------------------------------------------------

# PART A --- Create S3 storage

## 5. Step 1: Create an S3 bucket

1.  Sign in to AWS Console and search for **S3**.
2.  Choose **Create bucket**.
3.  Enter a unique name, for example `qb-web-data-84721`.
4.  Select Region `eu-central-1`.
5.  Keep Object Ownership as **ACLs disabled**.
6.  Keep **Block all public access** enabled.
7.  Keep default encryption (SSE-S3).
8.  Leave versioning disabled for this introductory lab.
9.  Choose **Create bucket**.

Keep the bucket private. EC2 will read the object through IAM
permissions; the object does not need a public URL.

## 6. Step 2: Create the product data file

On your computer, open Notepad or another plain-text editor. Create a
file named `products.json` and paste:

``` json
{
  "company": "Qualibytes IT Academy",
  "website": "AWS S3 Storage Lab",
  "products": [
    {"id": 101, "name": "Laptop", "price": 55000, "category": "Electronics"},
    {"id": 102, "name": "Mobile", "price": 18000, "category": "Electronics"},
    {"id": 103, "name": "Keyboard", "price": 1200, "category": "Accessories"},
    {"id": 104, "name": "Monitor", "price": 12500, "category": "Electronics"}
  ]
}
```

Save it as `products.json`, not `products.json.txt`.

## 7. Step 3: Upload the file

1.  Open your S3 bucket.
2.  Choose **Upload → Add files**.
3.  Select `products.json`.
4.  Choose **Upload**.
5.  Confirm the object appears in the bucket.

Expected bucket contents:

``` text
YOUR-BUCKET-NAME/
└── products.json
```

------------------------------------------------------------------------

# PART B --- IAM permissions for EC2

## 8. Step 4: Create a read-only S3 policy

1.  Open **IAM → Policies → Create policy**.
2.  Select the **JSON** editor.
3.  Paste this policy, replacing `qb-web-data-84721` with your exact
    bucket name:

``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::qb-web-data-84721"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::qb-web-data-84721/products.json"
    }
  ]
}
```

4.  Choose **Next**.
5.  Policy name: `QB-EC2-S3-Read-Policy`.
6.  Choose **Create policy**.

`ListBucket` permits listing bucket objects; `GetObject` permits reading
the specified JSON file. The policy does not grant upload or delete
permission. This is least-privilege access.

## 9. Step 5: Create an EC2 IAM role

1.  Open **IAM → Roles → Create role**.
2.  Trusted entity: **AWS service**.
3.  Use case: **EC2**.
4.  Choose **Next**.
5.  Select `QB-EC2-S3-Read-Policy`.
6.  Choose **Next**.
7.  Role name: `QB-EC2-S3-Role`.
8.  Choose **Create role**.

The role lets EC2 obtain temporary AWS credentials. Do not place access
keys or secret keys in the Python application.

------------------------------------------------------------------------

# PART C --- Build the first web server

## 10. Step 6: Launch an EC2 instance

1.  Open **EC2 → Launch instance**.
2.  Name: `QB-S3-WebServer`.
3.  AMI: **Amazon Linux 2023**.
4.  Instance type: a small suitable type, such as `t3.micro`, if
    available.
5.  Select an existing key pair or create one and securely save the
    private key.
6.  Select the lab VPC and a subnet with a suitable route for your
    chosen connection method.
7.  For initial SSH access, enable a public IP if required.
8.  Create/select a security group with inbound SSH port 22 from **My
    IP** only.
9.  Expand **Advanced details → IAM instance profile** and select
    `QB-EC2-S3-Role`.
10. Keep the default root EBS volume for this lab.
11. Launch the instance and wait for it to pass status checks.

Connect using EC2 Instance Connect if available, or SSH:

``` bash
ssh -i "YOUR-KEY.pem" ec2-user@YOUR-EC2-PUBLIC-IP
```

Replace the key filename and IP. The default Amazon Linux username is
`ec2-user`.

## 11. Step 7: Install Python and create the application directory

Run on EC2:

``` bash
sudo dnf update -y
sudo dnf install -y python3 python3-pip
python3 --version
pip3 --version
```

Create the directory and set ownership:

``` bash
sudo mkdir -p /opt/s3-web-app
sudo chown ec2-user:ec2-user /opt/s3-web-app
cd /opt/s3-web-app
```

Create a virtual environment and install the libraries:

``` bash
python3 -m venv venv
source venv/bin/activate
pip install flask boto3
```

Flask creates the website. Boto3 lets Python call AWS services such as
S3.

## 12. Step 8: Test EC2 access to S3

Install AWS CLI if needed:

``` bash
sudo dnf install -y awscli
```

Check the identity:

``` bash
aws sts get-caller-identity
```

It should show an assumed role associated with your EC2 instance. It
should not ask for long-term access keys.

List the bucket and read the file (replace the bucket name):

``` bash
aws s3 ls s3://YOUR-BUCKET-NAME/
aws s3 cp s3://YOUR-BUCKET-NAME/products.json -
```

The JSON content should appear in the terminal.

If you receive `AccessDenied`, check the attached role, bucket name,
policy ARN and object key.

## 13. Step 9: Create the Python application

From `/opt/s3-web-app`, create the file:

``` bash
vi app.py
```

In `vi`, press `i` to paste/type, then press `Esc`, type `:wq`, and
press Enter to save.

Paste this code and replace `YOUR-BUCKET-NAME` with your bucket name:

``` python
from flask import Flask, render_template
import boto3
import json

app = Flask(__name__)

BUCKET_NAME = "YOUR-BUCKET-NAME"
OBJECT_KEY = "products.json"
AWS_REGION = "eu-central-1"

# Boto3 automatically uses the EC2 instance role.
s3 = boto3.client("s3", region_name=AWS_REGION)

@app.route("/")
def home():
    try:
        response = s3.get_object(Bucket=BUCKET_NAME, Key=OBJECT_KEY)
        data = json.loads(response["Body"].read().decode("utf-8"))
        return render_template("index.html", data=data)
    except Exception as error:
        app.logger.exception("Could not retrieve data from S3")
        return f"Could not retrieve S3 data. Check application logs. Details: {error}", 500

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

## 14. Step 10: Create the HTML page

Create the templates folder and file:

``` bash
mkdir -p templates
vi templates/index.html
```

Paste:

``` html
<!DOCTYPE html>
<html>
<head>
    <title>Qualibytes Product Store</title>
    <style>
        body { font-family: Arial, sans-serif; background: #f2f5f9; margin: 40px; }
        .container { max-width: 850px; margin: auto; background: white; padding: 25px; border-radius: 10px; }
        h1 { color: #17365d; text-align: center; }
        table { width: 100%; border-collapse: collapse; }
        th, td { padding: 14px; border: 1px solid #ddd; text-align: left; }
        th { background: #17365d; color: white; }
        tr:nth-child(even) { background: #f2f5f9; }
    </style>
</head>
<body>
<div class="container">
    <h1>{{ data.company }}</h1>
    <h2>{{ data.website }}</h2>
    <h3>Available Products</h3>
    <table>
        <tr><th>ID</th><th>Product</th><th>Price (INR)</th><th>Category</th></tr>
        {% for product in data.products %}
        <tr>
            <td>{{ product.id }}</td>
            <td>{{ product.name }}</td>
            <td>₹{{ product.price }}</td>
            <td>{{ product.category }}</td>
        </tr>
        {% endfor %}
    </table>
    <p>Product data retrieved from Amazon S3.</p>
</div>
</body>
</html>
```

Save the file. The directory should now look like:

``` text
/opt/s3-web-app/
├── app.py
├── templates/
│   └── index.html
└── venv/
```

## 15. Step 11: Test the website on EC2

Start the app:

``` bash
cd /opt/s3-web-app
source venv/bin/activate
python app.py
```

In a second terminal connected to the same instance, run:

``` bash
curl http://localhost:5000
```

You should see HTML output. For an initial browser test only, you may
temporarily allow inbound TCP 5000 from **My IP** and visit:

``` text
http://YOUR-EC2-PUBLIC-IP:5000
```

Stop the foreground Flask process with `Ctrl+C` before continuing. The
final website will be accessed through the ALB on port 80, not directly
through the EC2 public IP.

## 16. Step 12: Configure the application to start automatically

Create a systemd service:

``` bash
sudo vi /etc/systemd/system/s3-web-app.service
```

Paste:

``` ini
[Unit]
Description=S3 Flask Web Application
After=network-online.target
Wants=network-online.target

[Service]
User=ec2-user
WorkingDirectory=/opt/s3-web-app
Environment="PATH=/opt/s3-web-app/venv/bin"
ExecStart=/opt/s3-web-app/venv/bin/python /opt/s3-web-app/app.py
Restart=always
RestartSec=5
StandardOutput=append:/var/log/s3-web-app.log
StandardError=append:/var/log/s3-web-app.log

[Install]
WantedBy=multi-user.target
```

Create the log file and grant the application user write access:

``` bash
sudo touch /var/log/s3-web-app.log
sudo chown ec2-user:ec2-user /var/log/s3-web-app.log
```

Enable and start the service:

``` bash
sudo systemctl daemon-reload
sudo systemctl enable s3-web-app
sudo systemctl start s3-web-app
sudo systemctl status s3-web-app
```

Test:

``` bash
curl http://localhost:5000
```

View logs:

``` bash
tail -f /var/log/s3-web-app.log
```

Press `Ctrl+C` to stop viewing logs; the service continues running.

------------------------------------------------------------------------

# PART D --- Load balancer and Auto Scaling

## 17. Step 13: Create an AMI

An AMI is a reusable image of the configured EC2 server.

1.  Open **EC2 → Instances**.
2.  Select `QB-S3-WebServer`.
3.  Choose **Actions → Image and templates → Create image**.
4.  Image name: `QB-S3-WebServer-AMI`.
5.  Keep the required root EBS volume.
6.  Choose **Create image**.
7.  Open **EC2 → AMIs** and wait for state **Available**.

Keep the original instance until the lab is complete.

## 18. Step 14: Configure network security groups

The internet-facing ALB will receive website traffic. EC2 should receive
application traffic only from the ALB.

For an internet-facing ALB, select public subnets in at least two
Availability Zones.

### EC2 web-server security group

Create or update the security group attached to the original EC2 instance and select the same group in the Launch Template. Name it `QB-Web-EC2-SG`.

**Inbound rules**

| Type | Protocol | Port | Source | Purpose |
|---|---|---:|---|---|
| SSH | TCP | 22 | Your public IP `/32` only | Optional administration from your computer. Never use `0.0.0.0/0`. |
| Custom TCP | TCP | 5000 | `QB-ALB-SG` (select the security group as source) | ALB forwards requests to the Flask app on EC2. |

Do not open port `5000` to the internet. Remove the temporary public port-5000 rule used for the initial direct-browser test. If you use Systems Manager Session Manager or an EC2 Instance Connect Endpoint, an inbound SSH rule may not be required.

**Outbound rules**

| Type | Protocol | Port | Destination | Purpose |
|---|---|---:|---|---|
| All traffic (default) | All | All | `0.0.0.0/0` | Simplest lab setting for package installation, AWS API access and updates. |

For a more restrictive design, allow the required destinations and ports. The instance needs HTTPS/TCP 443 connectivity to AWS services such as S3 and Systems Manager endpoints. An S3 Gateway VPC Endpoint can provide private S3 access, but does not provide connectivity to every other AWS service.

### ALB security group

Create `QB-ALB-SG` and attach it to the internet-facing Application Load Balancer.

**Inbound rules**

| Type | Protocol | Port | Source | Purpose |
|---|---|---:|---|---|
| HTTP | TCP | 80 | `0.0.0.0/0` | Allows visitors to open the website through the ALB DNS name. |
| HTTPS | TCP | 443 | `0.0.0.0/0` | Optional; add when an HTTPS listener and certificate are configured. |

This lab uses HTTP port 80. For production, configure HTTPS with an ACM certificate and redirect HTTP to HTTPS.

**Outbound rules**

| Type | Protocol | Port | Destination | Purpose |
|---|---|---:|---|---|
| Custom TCP | TCP | 5000 | `QB-Web-EC2-SG` | Allows ALB requests and health checks to reach Flask on EC2. |

The default ALB outbound rule (all traffic to `0.0.0.0/0`) also works for this lab. If you restrict outbound traffic, allow TCP 5000 to the EC2 security group. Security groups are stateful, so response traffic is automatically allowed.

### Port and traffic summary

| Traffic | Port | From → To | Required? |
|---|---:|---|---|
| Visitor → ALB | TCP 80 | Internet → `QB-ALB-SG` | Yes for this HTTP lab |
| Visitor → ALB | TCP 443 | Internet → `QB-ALB-SG` | Optional, for HTTPS |
| ALB → EC2 app | TCP 5000 | `QB-ALB-SG` → `QB-Web-EC2-SG` | Yes |
| Administrator → EC2 | TCP 22 | Your IP `/32` → `QB-Web-EC2-SG` | Optional, if using SSH |
| EC2 → AWS services | TCP 443 | EC2 → S3/AWS service endpoints | Yes for AWS API access/package downloads |
| ALB health check → EC2 | TCP 5000 | `QB-ALB-SG` → `QB-Web-EC2-SG` | Yes; same application port |

**Important:** The target group uses HTTP port `5000`, and its health-check path is `/`. The ALB listener receives traffic on port `80` and forwards it to the target group on port `5000`. Do not confuse the visitor-facing port with the Flask application port.

### Network connectivity note

EC2 must be able to reach S3. For a simple public-subnet lab, the
instance needs a suitable route and outbound connectivity. For
private-subnet EC2, use a suitable NAT path or preferably an S3 Gateway
VPC Endpoint. A Gateway Endpoint can provide private access to S3
without routing S3 traffic through an internet gateway or NAT Gateway.

## 19. Step 15: Create a target group

A target group is the set of EC2 servers that receive requests from the
ALB.

1.  Open **EC2 → Target Groups → Create target group**.
2.  Target type: **Instances**.
3.  Name: `QB-Web-TG`.
4.  Protocol: HTTP.
5.  Port: `5000`.
6.  Select the lab VPC.
7.  Health check protocol: HTTP.
8.  Health check path: `/`.
9.  Create the target group.

The Auto Scaling Group will register its instances with this target
group; you do not need to register the original server manually.

## 20. Step 16: Create an Application Load Balancer

1.  Open **EC2 → Load Balancers → Create load balancer**.
2.  Select **Application Load Balancer**.
3.  Name: `QB-Web-ALB`.
4.  Scheme: **Internet-facing**.
5.  IP address type: IPv4.
6.  Select the lab VPC.
7.  Select public subnets in at least two Availability Zones.
8.  Select security group `QB-ALB-SG`.
9.  Listener: HTTP port 80.
10. Forward the listener to `QB-Web-TG`.
11. Create the ALB and wait until it is **Active**.

## 21. Step 17: Create a Launch Template

A Launch Template tells Auto Scaling how to create each new EC2
instance.

1.  Open **EC2 → Launch Templates → Create launch template**.
2.  Name: `QB-S3-Web-LT`.
3.  AMI: `QB-S3-WebServer-AMI`.
4.  Instance type: the same suitable type used for the original server
    (for example `t3.micro`).
5.  Select the key pair if SSH access is needed.
6.  Select `QB-Web-EC2-SG` (the EC2 security group configured in Step 14).
7.  Expand **Advanced details → IAM instance profile**.
8.  Select `QB-EC2-S3-Role`.
9.  Create the Launch Template.

**Important:** Attaching a role to the original EC2 instance does not
automatically attach it to new instances. The IAM instance profile must
be set in the Launch Template.

## 22. Step 18: Create the Auto Scaling Group

1.  Open **EC2 → Auto Scaling Groups → Create Auto Scaling group**.

2.  Name: `QB-S3-Web-ASG`.

3.  Select Launch Template `QB-S3-Web-LT`.

4.  Select the lab VPC and suitable subnets in at least two Availability
    Zones.

5.  Set capacity:

      Setting              Value
      ------------------ -------
      Desired capacity         2
      Minimum capacity         1
      Maximum capacity         4

6.  Under load balancing, choose **Attach to an existing load
    balancer**.

7.  Select target group `QB-Web-TG`.

8.  Enable Elastic Load Balancing health checks.

9.  Set a health check grace period, for example 180 seconds.

10. Review and create the group.

Wait for two instances to launch.

Open **EC2 → Target Groups → QB-Web-TG → Targets** and wait until both
instances are **Healthy**.

### Test the website through the ALB

1.  Open **EC2 → Load Balancers**.
2.  Select `QB-Web-ALB`.
3.  Copy its DNS name.
4.  Open:

``` text
http://YOUR-ALB-DNS-NAME
```

The product website should appear. Use the ALB DNS name for the rest of
the lab.

## 23. Step 19: Create a CPU target-tracking policy

1.  Open **EC2 → Auto Scaling Groups**.
2.  Select `QB-S3-Web-ASG`.
3.  Open **Automatic scaling**.
4.  Choose **Create dynamic scaling policy**.
5.  Choose **Target tracking scaling policy**.
6.  Policy name: `QB-CPU-Target-Tracking`.
7.  Metric type: **Average CPU utilization**.
8.  Target value: `20`.
9.  Set instance warm-up to approximately 180 seconds.
10. Save the policy.

The policy tries to keep the group's average CPU utilization near 20% by
adjusting instance count. It does not guarantee an immediate launch
whenever one individual server crosses 20%.

Capacity remains minimum 1, desired 2 and maximum 4.

------------------------------------------------------------------------

# PART E --- Generate CPU load

## 24. Step 20: Install stress-ng

Connect to each of the two EC2 instances in the Auto Scaling Group.

Run:

``` bash
sudo dnf install -y stress-ng
```

If the package is unavailable in the configured repositories, use the
alternative command in the next step.

Check current CPU usage:

``` bash
top
```

Press `q` to exit.

## 25. Step 21: Generate CPU load

Run on the first EC2 instance:

``` bash
stress-ng --cpu 2 --timeout 300s
```

This runs two CPU workers for five minutes.

For a controlled load:

``` bash
stress-ng --cpu 2 --cpu-load 80 --timeout 300s
```

Repeat on the second EC2 instance if the Auto Scaling Group average is
not high enough.

### Alternative if stress-ng cannot be installed

Run:

``` bash
yes > /dev/null &
```

The terminal prints a process ID. Save it. Stop the process after the
test:

``` bash
kill YOUR-PROCESS-ID
```

Replace the placeholder with the actual process ID. Do not leave the
load process running.

## 26. Step 22: Observe Auto Scaling

While the CPU test runs, open these pages:

1.  **EC2 → Instances** --- look for another instance.
2.  **EC2 → Auto Scaling Groups → QB-S3-Web-ASG → Activity** --- see
    scale-out activity.
3.  **EC2 → Target Groups → QB-Web-TG → Targets** --- wait for the new
    instance to become healthy.
4.  **CloudWatch → Metrics → EC2** --- view CPU utilization.

Expected sequence:

  Stage                   Expected result
  ----------------------- --------------------------------------
  Before load             Two instances running
  CPU load starts         CPU increases
  Scaling condition met   ASG starts scale-out
  New instance launches   Third instance appears
  Health check passes     New instance becomes healthy
  Load stops              CPU decreases
  Scale-in evaluation     ASG may later remove excess capacity

**Testing notes:** - Metric collection, scaling decisions, EC2 launch
and target registration take time. - The target-tracking metric is the
**average CPU of the group**. Loading only one of two instances may not
raise the group average above 20%; generate load on both for a clearer
test. - `t3.micro` is burstable and CPU credits can affect the test. -
No new instance can launch once the group reaches its maximum of four. -
Do not expect exactly one instance to launch for every CPU increase.

------------------------------------------------------------------------

# PART F --- CloudWatch EC2 and application monitoring

## 27. Step 23: View EC2 metrics

EC2 sends basic metrics to CloudWatch automatically.

1.  Open **CloudWatch → Metrics → All metrics**.
2.  Choose **EC2**.
3.  Choose **Per-Instance Metrics**.
4.  Select `CPUUtilization` for your instances.
5.  View the graph.

Other useful metrics include `NetworkIn`, `NetworkOut`, `DiskReadBytes`,
`DiskWriteBytes` and `StatusCheckFailed`.

Basic monitoring generally provides five-minute metrics. Detailed
monitoring provides one-minute metrics and can incur additional charges.

To enable detailed monitoring, open **EC2 → Instances**, select an
instance, then choose **Actions → Monitor and troubleshoot → Manage
detailed monitoring**. Enable it if desired. Configure the Launch
Template too if newly launched instances should use detailed monitoring.

Detailed monitoring is optional for this lab.

## 28. Step 24: Give EC2 permission to publish logs

1.  Open **IAM → Roles**.
2.  Select `QB-EC2-S3-Role`.
3.  Choose **Add permissions → Attach policies**.
4.  Search for and attach AWS managed policy:

``` text
CloudWatchAgentServerPolicy
```

This allows the CloudWatch Agent to send logs and supported monitoring
data. The role is also used by instances launched from the Launch
Template.

## 29. Step 25: Install and configure the CloudWatch Agent

Connect to the original EC2 instance:

``` bash
sudo dnf install -y amazon-cloudwatch-agent
```

Create the configuration file:

``` bash
sudo vi /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

Paste:

``` json
{
  "agent": {
    "metrics_collection_interval": 60
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/s3-web-app.log",
            "log_group_name": "/qb/s3-web-app",
            "log_stream_name": "{instance_id}/application.log"
          }
        ]
      }
    }
  }
}
```

Save the file and start the agent:

``` bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
-a fetch-config \
-m ec2 \
-c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json \
-s
```

Check its status:

``` bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status
```

Open **CloudWatch → Logs → Log groups**. Find `/qb/s3-web-app`, open it
and inspect the log stream.

### Make log collection work on every Auto Scaling instance

The AMI used by the Launch Template must include the CloudWatch Agent,
its configuration and an enabled/running agent service.

Recommended sequence:

1.  Install and configure the agent on the original EC2.
2.  Attach `CloudWatchAgentServerPolicy` to the EC2 role.
3.  Start the agent and confirm the log group receives data.
4.  Create a new AMI after this configuration.
5.  Update the Launch Template to use the new AMI version.
6.  Update the Auto Scaling Group to use the new Launch Template
    version.
7.  Replace/refresh existing ASG instances so they use the updated
    image.

If you skip this, the original instance may send logs but newly launched
instances may not.

The configuration above collects the application log file. EC2 CPU
metrics are provided separately by EC2/CloudWatch.

------------------------------------------------------------------------

# PART G --- SNS email notifications

## 30. Step 26: Create an SNS topic

1.  Search for **Simple Notification Service (SNS)**.
2.  Open **Topics → Create topic**.
3.  Type: **Standard**.
4.  Name: `QB-EC2-CPU-Alert`.
5.  Choose **Create topic**.

## 31. Step 27: Subscribe an email address

1.  Open topic `QB-EC2-CPU-Alert`.
2.  Choose **Create subscription**.
3.  Protocol: **Email**.
4.  Endpoint: enter an email address you can access.
5.  Create the subscription.
6.  Open your inbox and find the AWS confirmation email.
7.  Click **Confirm subscription**.

The subscription must be confirmed before SNS can deliver emails. Check
the spam folder if the confirmation message is not visible.

## 32. Step 28: Create the CloudWatch CPU alarm

We will alarm on the **Auto Scaling Group average CPU**, not only one
individual EC2 instance.

1.  Open **CloudWatch → Alarms → Create alarm**.
2.  Choose **Select metric**.
3.  Find the EC2 metric for the Auto Scaling Group. Depending on the
    console layout, select the Auto Scaling Group dimension and its
    `CPUUtilization` metric.
4.  Select the metric for `QB-S3-Web-ASG`.
5.  Choose **Select metric**.

Configure:

  Setting               Value
  --------------------- ----------------
  Metric                CPUUtilization
  Statistic             Average
  Period                1 minute
  Threshold type        Static
  Condition             Greater than
  Threshold             20
  Evaluation periods    2
  Datapoints to alarm   2 out of 2

The alarm enters ALARM after the group average is above 20% for two
consecutive one-minute periods.

### Configure the email action

1.  Under notifications, select **In alarm**.
2.  Select existing SNS topic `QB-EC2-CPU-Alert`.
3.  Choose **Next**.
4.  Alarm name: `QB-ASG-CPU-Above-20`.
5.  Review and create the alarm.

### Understand the three separate jobs

  -----------------------------------------------------------------------
  Feature                             Job
  ----------------------------------- -----------------------------------
  Auto Scaling target-tracking policy Adds/removes EC2 instances to
                                      manage capacity

  CloudWatch alarm                    Detects CPU above the configured
                                      threshold

  SNS                                 Sends an email when the alarm
                                      changes state
  -----------------------------------------------------------------------

The target-tracking policy and the email alarm are separate
configurations. Both can use CPU metrics independently.

------------------------------------------------------------------------

# PART H --- Test the complete project

## 33. Step 29: Change data in S3 and refresh the website

Open the ALB DNS name:

``` text
http://YOUR-ALB-DNS-NAME
```

Confirm the product table appears.

### Test A: Change a product price

1.  Open S3 and your bucket.
2.  Select `products.json` and download it.
3.  Change the Laptop price from `55000` to `60000`.
4.  Save and upload the updated file, replacing the old object.
5.  Refresh the website.

The Laptop price should now display ₹60,000. This proves the website
reads its data from S3.

### Test B: Add a product

Add this object inside the `products` array:

``` json
{
  "id": 105,
  "name": "Headphones",
  "price": 2500,
  "category": "Accessories"
}
```

Upload the updated JSON file and refresh the website. The new product
should appear.

## 34. Step 30: Test Auto Scaling

1.  Confirm that two instances are healthy in the Auto Scaling Group.
2.  Connect to both instances.
3.  Run on each:

``` bash
stress-ng --cpu 2 --timeout 300s
```

4.  Watch the Auto Scaling Group Activity tab.
5.  Watch the EC2 Instances page.
6.  Watch the target group's health status.
7.  Wait for the new instance to become healthy.
8.  Stop the load or allow the five-minute test to finish.

Expected: when the group average reaches the target-tracking scaling
condition, Auto Scaling should attempt to add capacity, up to four
instances.

## 35. Step 31: Test the CloudWatch alarm and email

1.  Confirm the SNS subscription is confirmed.
2.  Open CloudWatch → Alarms.
3.  Select `QB-ASG-CPU-Above-20`.
4.  Generate CPU load on both initial instances.
5.  Wait for metric collection and alarm evaluation.
6.  Refresh the alarm page.
7.  When it enters **ALARM**, check the subscribed email inbox.

Expected: SNS sends an email when the alarm enters ALARM.

When CPU falls below the threshold for the configured evaluation
periods, the alarm returns to OK. You may optionally configure an SNS
notification for the OK state too.

If the alarm does not trigger, check the selected Auto Scaling Group
metric, average CPU, evaluation periods, metric data and SNS
subscription confirmation.

------------------------------------------------------------------------

# PART I --- Troubleshooting

## 36. Common problems

  -----------------------------------------------------------------------
  Problem                             What to check
  ----------------------------------- -----------------------------------
  S3 `AccessDenied`                   EC2 role attached; policy bucket
                                      name and object ARN correct

  `NoSuchBucket`                      Exact bucket name used in policy
                                      and Python file

  `NoSuchKey`                         Object key is exactly
                                      `products.json`

  No AWS credentials                  IAM role attached to EC2 and Launch
                                      Template

  Website does not open               ALB active, listener on 80, target
                                      healthy

  Target unhealthy                    Flask service running, health path
                                      `/`, EC2 SG allows port 5000 from
                                      ALB SG

  ALB returns 503                     No healthy targets registered

  New instance has no website         Correct AMI/Launch Template
                                      version; application service
                                      enabled

  Auto Scaling does not add capacity  Group average may not exceed
                                      target; check policy, metric, max
                                      capacity, quotas and launch errors

  CloudWatch logs missing             Agent status, IAM permissions, file
                                      path, configuration and
                                      connectivity

  SNS email missing                   Confirm subscription, alarm topic
                                      selection, spam folder

  SSH fails                           Public IP, route, security group
                                      source, key pair and username
                                      `ec2-user`
  -----------------------------------------------------------------------

Useful commands:

``` bash
# Application service
sudo systemctl status s3-web-app

# Restart application
sudo systemctl restart s3-web-app

# View application log
sudo tail -n 100 /var/log/s3-web-app.log

# Test application locally
curl http://localhost:5000

# Check EC2 identity
aws sts get-caller-identity

# Check S3 objects
aws s3 ls s3://YOUR-BUCKET-NAME/

# Check CloudWatch Agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status

# Check CPU
top
```

------------------------------------------------------------------------

# PART J --- Cleanup: remove resources to avoid future billing

## 37. Step 32: Delete the Auto Scaling Group first

Do this first. If the group remains active, it can launch replacement
instances.

1.  Open **EC2 → Auto Scaling Groups**.
2.  Select `QB-S3-Web-ASG`.
3.  Choose **Delete** and confirm.
4.  Wait for its managed EC2 instances to terminate.
5.  Open **EC2 → Instances** and verify those instances are terminated.

If the original manually launched instance `QB-S3-WebServer` is still
running, terminate it too when no longer required.

## 38. Step 33: Delete the ALB

1.  Open **EC2 → Load Balancers**.
2.  Select `QB-Web-ALB`.
3.  Choose **Delete** and confirm.

An ALB can incur charges while it exists.

## 39. Step 34: Delete the target group

1.  Open **EC2 → Target Groups**.
2.  Select `QB-Web-TG`.
3.  Choose **Delete**.

## 40. Step 35: Delete the Launch Template

1.  Open **EC2 → Launch Templates**.
2.  Select `QB-S3-Web-LT`.
3.  Choose **Delete template** and confirm.

## 41. Step 36: Deregister the AMI and delete snapshots

1.  Open **EC2 → AMIs**.
2.  Select `QB-S3-WebServer-AMI`.
3.  Choose **Deregister AMI**.
4.  Open **EC2 → Snapshots**.
5.  Identify snapshots created for this lab AMI.
6.  Delete those snapshots if they are no longer needed.

Deregistering an AMI does not automatically delete its EBS snapshots.
Snapshots can continue to incur storage charges.

## 42. Step 37: Delete CloudWatch resources

1.  Open **CloudWatch → Alarms**.
2.  Delete `QB-ASG-CPU-Above-20` and any other alarms created for the
    lab.
3.  Open **CloudWatch → Logs → Log groups**.
4.  Delete `/qb/s3-web-app`.
5.  Delete any custom dashboard created for this project.

## 43. Step 38: Delete the SNS topic

1.  Open **SNS → Topics**.
2.  Select `QB-EC2-CPU-Alert`.
3.  Choose **Delete** and confirm.

## 44. Step 39: Delete the S3 bucket

1.  Open **S3**.
2.  Select the lab bucket.
3.  Delete all objects.
4.  If versioning was enabled, delete all object versions and delete
    markers too.
5.  Delete the bucket.

A bucket cannot be deleted while it contains objects or versions.

## 45. Step 40: Delete the IAM role and policy

1.  Open **IAM → Roles**.
2.  Select `QB-EC2-S3-Role`.
3.  Detach policies if necessary and delete the role.
4.  Open **IAM → Policies**.
5.  Select `QB-EC2-S3-Read-Policy`.
6.  Delete it if it is no longer attached to another identity.

Do not delete AWS-managed policies.

## 46. Step 41: Check for leftover resources

  Resource                Cleanup action
  ----------------------- --------------------------------------------
  EC2 instances           Terminate lab instances
  EBS volumes             Delete unattached lab volumes
  EBS snapshots           Delete unused lab snapshots
  Elastic IPs             Release unused allocated addresses
  ALB                     Delete lab load balancer
  Target group            Delete lab target group
  NAT Gateway             Delete if created only for this lab
  VPC endpoints           Delete if created only for this lab
  S3 bucket               Empty and delete
  CloudWatch log groups   Delete lab log groups
  CloudWatch alarms       Delete lab alarms
  AMI                     Deregister and remove associated snapshots
  Launch Template         Delete

Do not delete shared VPCs, subnets, route tables, security groups, key
pairs or IAM policies used by other projects.

### Final billing verification

1.  Open **AWS Billing and Cost Management**.
2.  Open **Bills** or **Cost Explorer**.
3.  Review EC2, EBS, ELB, S3, CloudWatch and public IPv4 usage.
4.  Check again after billing usage data has updated.

Deleting resources prevents future usage charges for those resources; it
does not reverse charges already incurred.

------------------------------------------------------------------------

## 47. Student completion checklist

### S3 and IAM

-   [ ] Created a private S3 bucket.
-   [ ] Uploaded `products.json`.
-   [ ] Created the S3 read-only IAM policy.
-   [ ] Created the EC2 IAM role.
-   [ ] Attached the role to EC2.

### EC2 website

-   [ ] Launched Amazon Linux EC2.
-   [ ] Installed Python, Flask and Boto3.
-   [ ] Created `app.py` and `templates/index.html`.
-   [ ] Verified EC2 can read the S3 JSON file.
-   [ ] Opened the website successfully.
-   [ ] Configured the application as a systemd service.

### Load balancing and Auto Scaling

-   [ ] Created an AMI.
-   [ ] Created ALB and EC2 security groups.
-   [ ] Created a target group and Application Load Balancer.
-   [ ] Created a Launch Template and Auto Scaling Group.
-   [ ] Confirmed two healthy EC2 instances.
-   [ ] Opened the website through the ALB DNS name.
-   [ ] Created the CPU target-tracking policy.
-   [ ] Generated CPU load.
-   [ ] Observed scale-out and an additional EC2 instance.

### Monitoring and alerting

-   [ ] Viewed EC2 CPU metrics in CloudWatch.
-   [ ] Installed and started the CloudWatch Agent.
-   [ ] Verified application logs in CloudWatch Logs.
-   [ ] Created an SNS topic.
-   [ ] Confirmed the email subscription.
-   [ ] Created a CPU alarm with a 20% threshold.
-   [ ] Tested the alarm and received an email.

### Cleanup

-   [ ] Deleted the Auto Scaling Group.
-   [ ] Terminated the original EC2 instance.
-   [ ] Deleted the ALB and target group.
-   [ ] Deleted the Launch Template.
-   [ ] Deregistered the AMI and deleted its snapshots.
-   [ ] Deleted CloudWatch alarms and log groups.
-   [ ] Deleted the SNS topic.
-   [ ] Deleted the S3 bucket and its contents.
-   [ ] Deleted the lab IAM role and policy.
-   [ ] Checked for unattached EBS volumes, snapshots, Elastic IPs, NAT
    Gateways and other lab resources.
-   [ ] Reviewed AWS Billing.

## 48. Final outcome

You have built a website whose product data is stored privately in S3.
EC2 retrieves the data using an IAM role. An Application Load Balancer
distributes requests across the Auto Scaling Group. Auto Scaling
responds to CPU demand. CloudWatch collects metrics and application
logs, and SNS sends an email when the configured CPU alarm enters ALARM.

**Remember:** S3 stores data; EC2 runs the application; IAM secures
access; ALB distributes requests; Auto Scaling adjusts compute capacity;
CloudWatch monitors; SNS notifies.

## Official AWS documentation

-   [Amazon S3 User
    Guide](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)
-   [IAM roles for Amazon
    EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html)
-   [Application Load
    Balancers](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html)
-   [Amazon EC2 Auto
    Scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/what-is-amazon-ec2-auto-scaling.html)
-   [CloudWatch
    Agent](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent.html)
-   [CloudWatch
    alarms](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html)
-   [Amazon SNS email
    subscriptions](https://docs.aws.amazon.com/sns/latest/dg/sns-email-notifications.html)
