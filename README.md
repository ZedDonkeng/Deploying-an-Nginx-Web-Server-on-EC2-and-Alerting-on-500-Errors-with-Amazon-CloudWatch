# Deploying-an-Nginx-Web-Server-on-EC2-and-Alerting-on-500-Errors-with-Amazon-CloudWatch
Stream Nginx logs from EC2 to Amazon CloudWatch, create a Metric Filter for HTTP 500 errors, and get notified instantly via SNS email when your server hits a problem.



# AWS Labs

A collection of hands-on AWS labs covering infrastructure provisioning and observability. Each lab is self-contained and designed to be completed independently using the AWS Management Console and AWS CLI.

---

## Repository structure

```
aws-labs/
├── cloudformation/
│   └── vpc-stack.yaml          # CloudFormation template — Staging VPC + Nginx EC2
└── lab-guides/
    └── nginx-cloudwatch-lab.md # Lab guide — CloudWatch logging + 500 alerting
```

---

## Labs

### Lab 1 — Staging VPC and Nginx EC2 Instance (`cloudformation/vpc-stack.yaml`)

Provisions a complete staging environment in **us-east-1** using AWS CloudFormation. No key pairs required — access is handled via AWS Systems Manager Session Manager.

**What gets created:**

- A VPC (`172.16.0.0/16`) with 3 public and 3 private subnets across 3 availability zones
- An Internet Gateway and a NAT Gateway (in Public Subnet AZ A)
- Public and private route tables with appropriate default routes
- An IAM role and instance profile with the `AmazonSSMManagedInstanceCore` managed policy
- A security group allowing inbound HTTP (port 80) from anywhere
- A `t3.medium` EC2 instance running Amazon Linux 2 with Nginx pre-installed via UserData

**How to deploy:**

1. Open the [AWS CloudFormation console](https://console.aws.amazon.com/cloudformation)
2. Click **Create stack → With new resources**
3. Upload `cloudformation/vpc-stack.yaml`
4. Follow the prompts and click **Create stack**
5. Once the stack reaches `CREATE_COMPLETE`, find the EC2 public DNS in the **Outputs** tab

**Prerequisites:** An AWS account with permissions to create VPC, EC2, IAM, and CloudFormation resources.

---

### Lab 2 — Nginx Logs to CloudWatch with 500 Error Alerting (`lab-guides/nginx-cloudwatch-lab.md`)
**Prerequisites:** Lab 1 must be deployed and the `StagingWebServer` instance must be running and reachable via HTTP.

Configures the EC2 instance from Lab 1 to ship Nginx access and error logs to **Amazon CloudWatch Logs** using the CloudWatch agent. Then builds an alerting pipeline that detects HTTP 500 errors and sends email notifications via **Amazon SNS**.

**What you will do:**

1. Install and configure the CloudWatch agent on the Nginx EC2 instance
2. Stream `/var/log/nginx/access.log` and `/var/log/nginx/error.log` to CloudWatch Log Groups
3. Deliberately trigger HTTP 500 errors to populate the logs
4. Create a **Metric Filter** matching `%\b500\b%` in the access log stream
5. Create a **CloudWatch Alarm** (Sum > 0, 1-minute period) on the 500 metric
6. Wire the alarm to an **SNS topic** (`500-Alerts`) for real-time email notifications

Lab2 step by step : [lab-guides-nginx-cloudwatch-lab](https://github.com/ZedDonkeng/Deploying-an-Nginx-Web-Server-on-EC2-and-Alerting-on-500-Errors-with-Amazon-CloudWatch/blob/main/nginx-cloudwatch-lab.md)

---

## Requirements

| Tool | Purpose |
|---|---|
| AWS account | Running all resources |
| IAM permissions | CloudFormation, EC2, VPC, IAM, CloudWatch, SNS |
| Browser | Accessing the Nginx page and AWS console |
| Email address | Receiving SNS alarm notifications |

---

## Cost note

The resources in these labs are **not free tier eligible** in all cases (NAT Gateway, `t3.medium`). Remember to delete the CloudFormation stack when you are done to avoid ongoing charges.

```bash
# Delete the stack via AWS CLI
aws cloudformation delete-stack --stack-name <your-stack-name> --region us-east-1
```

---

## Author

Built as a pair of companion labs — deploy the infrastructure with Lab 1, then instrument it with Lab 2.
