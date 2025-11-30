# Host a Static Website on AWS

This project demonstrates how to deploy and host a static HTML website on AWS using a highly available, secure, and scalable VPC architecture. The website is served by Apache running on EC2 instances in private subnets, behind an Application Load Balancer (ALB) and managed by an Auto Scaling Group (ASG). DNS and TLS are configured with Route 53 and AWS Certificate Manager (ACM), and scaling activities are monitored with SNS notifications.

## Architecture Overview

![Alt text](/Architecture.pdf)


---

### Deployment Steps (High Level)

1) Network Foundation
   
   -	Create a VPC with:
   -	2 Public Subnets (different AZs)
   -	2 Private App Subnets (different AZs)
   -	2 Private Data Subnets (different AZs)
   -	Attach an Internet Gateway.
   -	Create public route tables routing `0.0.0.0/0 → IGW`.
   -	Create NAT Gateway(s) in public subnets with Elastic IP.
   -	Create private route tables routing `0.0.0.0/0 → NAT Gateway`.

3) Security

   Set ALB Security Group:
	-	Allow inbound `80/443` from `0.0.0.0/0`
	-	Allow outbound to EC2 security group

	  Set EC2 Security Group:
	-	Allow inbound `80` only from `ALB SG`
	-	Allow outbound `0.0.0.0/0` (via NAT)

4) Compute + Scaling
	•	Create a Launch Template using Amazon Linux 2 (or AL2023).
	•	Add the user-data script below to bootstrap Apache and pull your site from GitHub.
	•	Create an Auto Scaling Group across private app subnets.
	•	Attach ASG to a Target Group (HTTP health checks).

5) Load Balancing
	•	Deploy an Application Load Balancer in public subnets.
	•	Listener rules:
	•	Port 80 → redirect to 443
	•	Port 443 → forward to Target Group

6) HTTPS + DNS
	•	Request an ACM certificate for your domain.
	•	Validate via DNS in Route 53.
	•	Create Route 53 A/AAAA alias record pointing domain → ALB.

7) Monitoring / Alerts
	•	Configure SNS topic + subscription.
	•	Add ASG notifications:
	•	instance launch
	•	instance terminate
	•	scaling events

---

### EC2 Bootstrap Script (User Data)

This script installs Apache, pulls the static site from GitHub, and serves it.
Note: The script below includes corrected package/service names and URL.

``` bash
#!/bin/bash
# Switch to root for admin privileges
sudo su

# Update installed packages
yum update -y

# Install Apache HTTP Server
yum install -y httpd

# Enable Apache at boot and start it
systemctl enable httpd
systemctl start httpd

# Install Git
yum install -y git

# Move into Apache web root
cd /var/www/html

# Clone repo and copy site content
git clone https://github.com/aosnotes77/host-a-static-website-on-aws.git
cp -R host-a-static-website-on-aws/. /var/www/html/

# Cleanup
rm -rf host-a-static-website-on-aws
```

---

### Repository Structure (Example)

.
├── diagram/                    # Architecture diagram(s)
├── scripts/
│   └── user-data.sh            # EC2 bootstrap script
├── website/                    # Static HTML/CSS/JS content
│   ├── index.html
│   ├── styles.css
│   └── assets/
└── README.md


---

### How to Validate the Deployment
	1.	Confirm instances are in private subnets and passing health checks.
	2.	Verify ALB target group shows healthy instances.
	3.	Visit the domain:
	•	http://yourdomain.com should redirect to HTTPS.
	•	https://yourdomain.com should load the static site.
	4.	Trigger scaling (optional):
	•	Increase load or reduce ASG desired capacity and watch SNS notifications.

---

### Troubleshooting

Targets show unhealthy
	•	Ensure EC2 SG allows inbound HTTP only from ALB SG.
	•	Confirm Apache is installed and running:

systemctl status httpd


	•	Ensure health check path matches your site (e.g., /index.html).

Instances can’t pull from GitHub
	•	Check private route table points to NAT Gateway.
	•	Confirm NAT Gateway is in a public subnet with route to IGW.

HTTPS not working
	•	Verify ACM cert is issued and attached to ALB listener.
	•	Confirm Route 53 validation CNAME exists.

---

### Improvements / Next Steps
	•	Add a CI/CD pipeline (GitHub Actions → CodeDeploy or SSM) for automatic updates.
	•	Store static assets in S3 + CloudFront for cheaper global delivery.
	•	Enable WAF on the ALB for added security.
	•	Enable CloudWatch alarms for CPU/network scaling policies.

---

License

This project is open-source and available under the MIT License (or update as preferred).
