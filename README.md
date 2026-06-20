# AWS Automated Disaster Recovery & DNS Failover using Lambda and Route 53

## 📌 Project Overview (Project #42)
This project demonstrates an enterprise-grade Automated Disaster Recovery (DR) and Failover mechanism using serverless AWS resources. A custom automation script monitors a primary application server, and upon detecting a critical downtime event, programmatically mutates DNS routing records via Route 53 to redirect incoming traffic to a highly resilient standby S3 environment.

## 🏗️ Architecture Components
* **Primary Application:** AWS EC2 Instance (Ubuntu LTS) running an Nginx Web Server.
* **Backup Application:** AWS S3 Bucket configured for Static Website Hosting.
* **DNS & Routing:** AWS Route 53 Hosted Zone managing core domain endpoints.
* **Automation Brain:** AWS Lambda Function running a custom Python script powered by `boto3` and `urllib3`.
* **Security Configuration:** Secure IAM execution roles with targeted Route 53 update access (`AmazonRoute53FullAccess`).

---

## 🚀 Step-by-Step Implementation

### Step 1: Standby Environment Setup (S3)
1. Created an S3 Bucket with Public Access enabled.
2. Uploaded a modern, dynamic, enterprise-branded `index.html` maintenance dashboard.
3. Enabled **Static website hosting** and recorded the bucket's global endpoint URL.
4. Applied a rigorous Bucket Policy allowing global `s3:GetObject` execution.

### Step 2: Primary Infrastructure Setup (EC2)
1. Provisioned a `t2.micro` Ubuntu instance wrapped in a customized Security Group (`web-server-sg`).
2. Configured Security Group Inbound rules to allow traffic over port `22` (SSH) and port `80` (HTTP).
3. Injected a automated shell script into the EC2 **User Data** to handle system upgrades, install Nginx dynamically, and launch the primary operational homepage.

### Step 3: DNS Orchestration (Route 53)
1. Maintained a Private Hosted Zone representing the organizational routing network.
2. Mapped an `A` record pointing traffic directly to the EC2 Public IPv4 address.
3. Adjusted the Time-to-Live (TTL) down to **60 seconds** to support instantaneous failover shifts.

### Step 4: The Failover Automation Engine (Lambda)
1. Created an AWS Lambda function running under **Python 3.12**.
2. Adjusted the standard IAM Execution Role permissions to handle Route 53 record modifications.
3. Wrote a deterministic validation script that runs real-time health evaluations against the Primary Application Server.

---

## 🧠 Automation Logic Script (`lambda_function.py`)

```python
import json
import boto3
import urllib3

route53 = boto3.client('route53')

# Infrastructure Variables
HOSTED_ZONE_ID = "YOUR_HOSTED_ZONE_ID"  
DOMAIN_NAME = "www.my-enterprise-app.local"  
PRIMARY_IP = "YOUR_EC2_PUBLIC_IP"        
BACKUP_S3_ENDPOINT = "YOUR_S3_STATIC_URL_WITHOUT_HTTP" 

def lambda_handler(event, context):
    http = urllib3.PoolManager(timeout=3.0)
    try:
        # Check primary infrastructure health
        response = http.request('GET', f"http://{PRIMARY_IP}")
        if response.status == 200:
            print("🟢 Primary Server is Healthy and Online!")
            return {"status": "Healthy"}
    except Exception as e:
        print("🔴 Primary Server is DOWN! Activating Disaster Recovery Failover...")
        # Automate DNS Upsert modification
        route53.change_resource_record_sets(
            HostedZoneId=HOSTED_ZONE_ID,
            ChangeBatch={
                'Comment': 'Automated failover triggering due to EC2 downtime.',
                'Changes': [
                    {
                        'Action': 'UPSERT', 
                        'ResourceRecordSet': {
                            'Name': DOMAIN_NAME,
                            'Type': 'CNAME', 
                            'TTL': 60,
                            'ResourceRecords': [{'Value': BACKUP_S3_ENDPOINT}]
                        }
                    }
                ]
            }
        )
        print("🔄 Route 53 DNS updated successfully. Traffic routed to S3 Backup Website!")
    return {"status": "Failover Executed"}
