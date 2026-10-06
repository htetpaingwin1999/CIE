# 🌏 AWS Geolocation Routing Project

## 📌 Overview

This project demonstrates how to use **Amazon Route 53 Geolocation Routing** to direct users to different AWS regions based on their geographic location.

The application is deployed in two AWS regions:

- 🇯🇵 **Tokyo Region** — `ap-northeast-1`
- 🇸🇬 **Singapore Region** — `ap-southeast-1`

Both environments are accessed using the same domain:

```text
https://dashboard.htetpaingwincloudlab.xyz
```

Route 53 determines the geographic location of the DNS request and routes users to the corresponding regional Application Load Balancer.

---

## 🎯 Routing Result

| User Location | Route 53 Destination | Application |
|---|---|---|
| 🇯🇵 Japan | Tokyo Dashboard ALB | Dashboard From Tokyo |
| 🇸🇬 Singapore | Singapore Dashboard ALB | Dashboard From Singapore |

---

# 🏗️ Architecture

![AWS Geolocation Routing Architecture](./architecture/aws-geolocation-routing-architecture.png)

### Traffic Flow

```text
                         User
                          │
                          ▼
        dashboard.htetpaingwincloudlab.xyz
                          │
                          ▼
                   Amazon Route 53
                Geolocation Routing
                   /             \
                  /               \
             🇯🇵 Japan          🇸🇬 Singapore
                │                    │
                ▼                    ▼
        Tokyo Dashboard ALB   Singapore Dashboard ALB
             HTTPS 443              HTTPS 443
                │                    │
                ▼                    ▼
        Dashboard EC2 :8888   Dashboard EC2 :8888
                │                    │
                ▼                    ▼
        Internal Counting ALB Internal Counting ALB
              HTTP 80               HTTP 80
                │                    │
                ▼                    ▼
        Counting EC2 :9902    Counting EC2 :9902
```

---

# 🌐 Infrastructure Setup

## 🇯🇵 Tokyo Region

```text
Region:              ap-northeast-1
Availability Zones:  ap-northeast-1a, ap-northeast-1c
Dashboard ALB:       Internet-facing
Counting ALB:        Internal
Dashboard Port:      8888
Counting Port:       9902
```

## 🇸🇬 Singapore Region

```text
Region:              ap-southeast-1
Availability Zones:  ap-southeast-1a, ap-southeast-1c
Dashboard ALB:       Internet-facing
Counting ALB:        Internal
Dashboard Port:      8888
Counting Port:       9902
```

---

# 🇯🇵 Tokyo Region Setup

## 1. Network Resources

| Resource | Name |
|---|---|
| VPC | `JP-VPC` |
| Public Subnet 1 | `JP-Public-1a` |
| Public Subnet 2 | `JP-Public-1c` |
| Private Subnet 1 | `JP-Private-1a` |
| Private Subnet 2 | `JP-Private-1c` |
| Internet Gateway | `JP-IGW` |
| Public Route Table | `JP-Public-RT` |
| Bastion Instance | `JP-Bastion` |
| Dashboard Instance | `JP-Dashboard-Instance` |
| Counting Instance | `JP-Counting-Instance` |
| Dashboard ALB | `JP-Dashboard-ALB` |
| Counting ALB | `JP-Counting-ALB` |
| Dashboard Target Group | `JP-Dashboard-Target-Group` |
| Counting Target Group | `JP-Counting-Target-Group` |

### Network Configuration

```text
VPC CIDR:
10.10.0.0/16

Public Subnets:
10.10.1.0/24
10.10.2.0/24

Private Subnets:
10.10.11.0/24
10.10.12.0/24
```

### Public Route

```text
Destination: 0.0.0.0/0
Target:      JP-IGW
```

### Screenshots

![Tokyo VPC](./screenshots/tokyo/jp-vpc.png)

![Tokyo Subnets](./screenshots/tokyo/jp-subnets.png)

![Tokyo Route Table](./screenshots/tokyo/jp-route-table.png)

---

# 🔐 Tokyo Security Groups

## 2. Bastion Security Group

```text
Security Group:
JP-Bastion-SG

Inbound:
SSH 22 <- My IP

Outbound:
SSH 22 -> JP-Dashboard-Instance-SG
SSH 22 -> JP-Counting-Instance-SG
```

![Tokyo Bastion Security Group]
(./screenshots/tokyo/jp-bastion-sg-inbound.png)
(./screenshots/tokyo/jp-bastion-sg-outbound.png)

---

## 3. Dashboard ALB Security Group

```text
Security Group:
JP-Dashboard-ALB-SG

Inbound:
HTTP 80   <- 0.0.0.0/0
HTTPS 443 <- 0.0.0.0/0

Outbound:
TCP 8888 -> JP-Dashboard-Instance-SG
```

![Tokyo Dashboard ALB Security Group]
(./screenshots/tokyo/jp-dashboard-alb-sg-inbound.png)
(./screenshots/tokyo/jp-dashboard-alb-sg-outbound.png)

---

## 4. Dashboard Instance Security Group

```text
Security Group:
JP-Dashboard-Instance-SG

Inbound:
TCP 8888 <- JP-Dashboard-ALB-SG
SSH 22   <- JP-Bastion-SG

Outbound:
HTTP 80 -> JP-Counting-ALB-SG
```

![Tokyo Dashboard Instance Security Group]
(./screenshots/tokyo/jp-dashboard-instance-sg-inbound.png)
(./screenshots/tokyo/jp-dashboard-instance-sg-outbound.png)

---

## 5. Counting ALB Security Group

```text
Security Group:
JP-Counting-ALB-SG

Inbound:
HTTP 80 <- JP-Dashboard-Instance-SG

Outbound:
TCP 9902 -> JP-Counting-Instance-SG
```

![Tokyo Counting ALB Security Group]
(./screenshots/tokyo/jp-counting-alb-sg-inbound.png)
(./screenshots/tokyo/jp-counting-alb-sg-outbound.png)

---

## 6. Counting Instance Security Group

```text
Security Group:
JP-Counting-Instance-SG

Inbound:
TCP 9902 <- JP-Counting-ALB-SG
SSH 22   <- JP-Bastion-SG

Outbound:
All Traffic -> 0.0.0.0/0
```

![Tokyo Counting Instance Security Group]
(./screenshots/tokyo/jp-counting-instance-sg-inbound.png)
(./screenshots/tokyo/jp-counting-instance-sg-outbound.png)

---
## 7. Counting Instance Setup

Create the Counting EC2 instance in the Tokyo private subnet.

### Instance Configuration

```text
Instance Name:
JP-Counting-Instance

Region:
ap-northeast-1

VPC:
JP-VPC

Subnet:
JP-Private-1a

Public IP:
Disabled

Security Group:
JP-Counting-Instance-SG

Application Port:
9902
```

![Tokyo Counting Instance]
(./screenshots/tokyo/jp-counting-instance-1a.png)
(./screenshots/tokyo/jp-counting-instance-1c.png)

---

## 8. Counting Service Deployment

### Build the Linux Binary

```bash
GOOS=linux GOARCH=amd64 go build -o counting-service-linux-amd64 .
```

### Local → Bastion

```bash
scp -i <key.pem> \
counting-service-linux-amd64 \
ubuntu@<JP_BASTION_PUBLIC_IP>:/home/ubuntu/
```

### Bastion → Counting Instance

```bash
scp -i <key.pem> \
counting-service-linux-amd64 \
ubuntu@<JP_COUNTING_PRIVATE_IP>:/home/ubuntu/
```

### Prepare Application Directory

```bash
sudo mkdir -p /opt/counting-service
```

```bash
sudo mv /home/ubuntu/counting-service-linux-amd64 \
/opt/counting-service/counting-service
```

```bash
sudo chmod +x /opt/counting-service/counting-service
```

### Create systemd Service

```bash
sudo nano /etc/systemd/system/counting-service.service
```

```ini
[Unit]
Description=Counting Service
After=network.target

[Service]
Type=simple
User=ubuntu
Group=ubuntu
WorkingDirectory=/opt/counting-service

Environment=PORT=9902

ExecStart=/opt/counting-service/counting-service

Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### Start the Counting Service

```bash
sudo systemctl daemon-reload
sudo systemctl enable counting-service
sudo systemctl start counting-service
sudo systemctl status counting-service
```

---

## 9. Counting Target Group

Create a target group for the Counting service.

```text
Target Type:
Instances

Protocol:
HTTP

Port:
9902

VPC:
JP-VPC

Health Check Protocol:
HTTP

Health Check Path:
/
```

Register:

```text
JP-Counting-Instance-1a & JP-Counting-Instance-1c
```

The target should become:

```text
Healthy
```

![Tokyo Counting Target Group]
(./screenshots/tokyo/jp-counting-target-group.png)

---

## 10. Counting Internal Application Load Balancer

Create an internal Application Load Balancer for the Counting service.

```text
Name:
JP-Counting-Internal-ALB

Scheme:
Internal

IP Address Type:
IPv4

VPC:
JP-VPC

Subnets:
JP-Private-1a
JP-Private-1c

Security Group:
JP-Counting-ALB-SG
```

### Listener

```text
Protocol:
HTTP

Port:
80

Default Action:
Forward to JP-Counting-Target-Group
```

Traffic flow:

```text
Dashboard Instance
        |
        | HTTP : 80
        v
JP-Counting-ALB
        |
        | TCP : 9902
        v
JP-Counting-Instance
```

![Tokyo Counting ALB]
(./screenshots/tokyo/jp-counting-alb.png)

![Tokyo Counting ALB Listener]
(./screenshots/tokyo/jp-counting-alb-resource-map.png)

---

## 11. Dashboard Instance Setup

Create the Dashboard EC2 instance in the Tokyo private subnet.

### Instance Configuration

```text
Instance Name:
JP-Dashboard-Instance

Region:
ap-northeast-1

VPC:
JP-VPC

Subnet:
JP-Private-1a

Public IP:
Disabled

Security Group:
JP-Dashboard-Instance-SG

Application Port:
8888
```

![Tokyo Dashboard Instance]
(./screenshots/tokyo/jp-dashboard-instance-1a.png)
(./screenshots/tokyo/jp-dashboard-instance-1b.png)
---

## 12. Dashboard Service Deployment

### Build the Linux Binary

```bash
GOOS=linux GOARCH=amd64 go build -o dashboard-service-linux-amd64 .
```

### Local → Bastion

```bash
scp -i <key.pem> \
dashboard-service-linux-amd64 \
ubuntu@<JP_BASTION_PUBLIC_IP>:/home/ubuntu/
```

### Bastion → Dashboard Instance

```bash
scp -i <key.pem> \
dashboard-service-linux-amd64 \
ubuntu@<JP_DASHBOARD_PRIVATE_IP>:/home/ubuntu/
```

### Prepare Application Directory

```bash
sudo mkdir -p /opt/dashboard-service
```

```bash
sudo mv /home/ubuntu/dashboard-service-linux-amd64 \
/opt/dashboard-service/dashboard-service
```

```bash
sudo chmod +x /opt/dashboard-service/dashboard-service
```

### Create systemd Service

```bash
sudo nano /etc/systemd/system/dashboard-service.service
```

```ini
[Unit]
Description=Dashboard Service
After=network.target

[Service]
Type=simple
User=ubuntu
Group=ubuntu
WorkingDirectory=/opt/dashboard-service

Environment=PORT=8888
Environment=COUNTING_SERVICE_URL=http://<JP_COUNTING_INTERNAL_ALB_DNS>/

ExecStart=/opt/dashboard-service/dashboard-service

Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Replace:

```text
<JP_COUNTING_INTERNAL_ALB_DNS>
```

with the DNS name of `JP-Counting-ALB`.

Example:

```text
Environment=COUNTING_SERVICE_URL=http://internal-jp-counting-alb-xxxxxxxx.ap-northeast-1.elb.amazonaws.com/
```

### Start the Dashboard Service

```bash
sudo systemctl daemon-reload
sudo systemctl enable dashboard-service
sudo systemctl start dashboard-service
sudo systemctl status dashboard-service
```

---

## 13. Dashboard Target Group

Create a target group for the Dashboard service.

```text
Target Type:
Instances

Protocol:
HTTP

Port:
8888

VPC:
JP-VPC

Health Check Protocol:
HTTP

Health Check Path:
/
```

Register:

```text
JP-Dashboard-Instance-1a & JP-Dashboard-Instance-1c
```

The target should become:

```text
Healthy
```

![Tokyo Dashboard Target Group]
(./screenshots/tokyo/jp-dashboard-target-group.png)



---

## 14. Dashboard Internet-Facing Application Load Balancer

Create the public Application Load Balancer for the Dashboard application.

```text
Name:
JP-Dashboard-ALB

Scheme:
Internet-facing

IP Address Type:
IPv4

VPC:
JP-VPC

Subnets:
JP-Public-1a
JP-Public-1c

Security Group:
JP-Dashboard-ALB-SG
```

### HTTP Listener

```text
Protocol:
HTTPs

Port:
443

Default Action:
Forward to JP-Dashboard-Target-Group

Protocol:
HTTP

Port:
80

Redirect to HTTPS://#{host}:443/#{path}?#{query}
```

At this stage, the traffic flow is:

```text
Internet
   |
   | HTTP : 80
   v
JP-Dashboard-ALB
   |
   | TCP : 8888
   v
JP-Dashboard-Instance
   |
   | HTTP : 80
   v
JP-Counting-ALB
   |
   | TCP : 9902
   v
JP-Counting-Instance
```

![Tokyo Dashboard ALB]
(./screenshots/tokyo/jp-dashboard-alb.png)

![Tokyo Dashboard ALB Listener](./screenshots/tokyo/jp-dashboard-alb-resource-map.png)

---

## 15. Verify Service Status

![Tokyo Dashboard Test](./screenshots/tokyo/jp-dashboard-test.png)

---

# 🇸🇬 Singapore Region Setup

The same VPC, EC2, Security Group, Target Group, Application Load Balancer, and service configuration as the Tokyo region was created in the Singapore region (`ap-southeast-1`).


# 🔒 SSL/TLS Certificate

AWS Certificate Manager (ACM) is used to secure the Dashboard domain.

Domain:

```text
dashboard.htetpaingwincloudlab.xyz
```

Because ACM certificates are regional, certificates were created separately in Tokyo and Singapore.

---

## 16. Tokyo ACM Certificate

```text
Region:
ap-northeast-1

Domain:
dashboard.htetpaingwincloudlab.xyz

Certificate Type:
Public Certificate

Validation:
DNS Validation
```

![Tokyo ACM Certificate]
(./screenshots/tokyo/tokyo-certificate.png)

---

## 17. Singapore ACM Certificate

```text
Region:
ap-southeast-1

Domain:
dashboard.htetpaingwincloudlab.xyz

Certificate Type:
Public Certificate

Validation:
DNS Validation
```

![Singapore ACM Certificate]
(./screenshots/singapore/singapore-certificate.png)

---

# 🔁 HTTP to HTTPS Redirect

The Dashboard ALBs are configured with two listeners.

```text
HTTP : 80
      │
      ▼
Redirect
      │
      ▼
HTTPS : 443
      │
      ▼
Dashboard Target Group : 8888
```

### HTTP Listener

```text
Protocol:
HTTP

Port:
80

Action:
Redirect to HTTPS

Redirect Port:
443

Status Code:
HTTP 301
```

### HTTPS Listener

```text
Protocol:
HTTPS

Port:
443

Action:
Forward to Dashboard Target Group
```

![HTTPS Listener](./screenshots/certificate/https-listener.png)

---

# 🌍 Route 53 Configuration

## 18. Public Hosted Zone

Hosted Zone:

```text
htetpaingwincloudlab.xyz
```

Type:

```text
Public Hosted Zone
```

![Route 53 Hosted Zone]
(./screenshots/route53/route53-hosted-zone.png)

---

# 🔗 Namecheap DNS Delegation

The domain was registered using Namecheap.

Namecheap nameservers were changed from Namecheap BasicDNS to the Route 53 authoritative nameservers.

Example:

```text
ns-324.awsdns-40.com
ns-1300.awsdns-34.org
ns-879.awsdns-45.net
ns-1950.awsdns-51.co.uk
```

Verify DNS delegation:

```bash
dig NS htetpaingwincloudlab.xyz
```

Expected result:

```text
status: NOERROR
ANSWER: 4
```


# 🌏 Route 53 Geolocation Routing

The original Simple Routing record was replaced with two Geolocation Routing records.

---

## 19. Japan Geolocation Record

```text
Record Name:
dashboard.htetpaingwincloudlab.xyz

Record Type:
A

Alias:
Yes

Routing Policy:
Geolocation

Location:
Japan

Target:
Tokyo Dashboard ALB

Set ID:
JP
```
---

## 20. Singapore Geolocation Record

```text
Record Name:
dashboard.htetpaingwincloudlab.xyz

Record Type:
A

Alias:
Yes

Routing Policy:
Geolocation

Location:
Singapore

Target:
Singapore Dashboard ALB

Set ID:
SG
```

![Japan & Singapore Geolocation Record]
(./screenshots/route53/geolocation-record.png)

---
# 🇯🇵 Geolocation Test — Japan

VPN Location:

```text
Japan
```

URL:

```text
https://dashboard.htetpaingwincloudlab.xyz
```

Expected Result:

```text
Dashboard From Tokyo
```

![Japan VPN]
(./screenshots/testing/japan-vpn.png)

![Dashboard From Tokyo]
(./screenshots/testing/dashboard-from-tokyo.png)

---

# 🇸🇬 Geolocation Test — Singapore

VPN Location:

```text
Singapore
```

URL:

```text
https://dashboard.htetpaingwincloudlab.xyz
```

Expected Result:

```text
Dashboard From Singapore
```

![Singapore VPN]
(./screenshots/testing/singapore-vpn.png)

![Dashboard From Singapore]
(./screenshots/testing/dashboard-from-singapore.png)

---

# ✅ Final Result

The same domain successfully routes users to different AWS regions according to their geographic location.

### Japan

```text
Japan User
     ↓
Amazon Route 53
     ↓
Geolocation: Japan
     ↓
Tokyo Dashboard ALB
     ↓
Tokyo Dashboard EC2
     ↓
Tokyo Counting ALB
     ↓
Tokyo Counting EC2
```

### Singapore

```text
Singapore User
     ↓
Amazon Route 53
     ↓
Geolocation: Singapore
     ↓
Singapore Dashboard ALB
     ↓
Singapore Dashboard EC2
     ↓
Singapore Counting ALB
     ↓
Singapore Counting EC2
```

---

# 🏁 Project Outcome

The project successfully implemented:

- ✅ Multi-region deployment
- ✅ Tokyo and Singapore environments
- ✅ Separate VPC infrastructure
- ✅ Public Dashboard ALBs
- ✅ Internal Counting ALBs
- ✅ Bastion access to private instances
- ✅ Dashboard application on port `8888`
- ✅ Counting application on port `9902`
- ✅ AWS Certificate Manager
- ✅ HTTPS using port `443`
- ✅ HTTP to HTTPS redirection
- ✅ Route 53 Public Hosted Zone
- ✅ Namecheap DNS delegation
- ✅ Route 53 Geolocation Routing
- ✅ Japan → Tokyo routing
- ✅ Singapore → Singapore routing
- ✅ VPN-based geolocation testing

---

## 📁 Repository Structure

```text
geolocation-routing/
│
├── README.md
│
├── architecture/
│   └── aws-geolocation-routing-architecture.png
│
└── screenshots/
    │
    ├── tokyo/
    │   ├── jp-vpc.png
    │   ├── jp-subnets.png
    │   ├── jp-route-table.png
    │   ├── jp-bastion-sg.png
    │   ├── jp-dashboard-alb-sg.png
    │   ├── jp-dashboard-instance-sg.png
    │   ├── jp-counting-alb-sg.png
    │   ├── jp-counting-instance-sg.png
    │   ├── jp-ec2-instances.png
    │   ├── jp-dashboard-service.png
    │   ├── jp-counting-service.png
    │   ├── jp-dashboard-alb.png
    │   ├── jp-dashboard-target-group.png
    │   ├── jp-counting-alb.png
    │   ├── jp-counting-target-group.png
    │   └── jp-dashboard-test.png
    │
    ├── singapore/
    │   ├── sg-vpc.png
    │   ├── sg-subnets.png
    │   ├── sg-ec2-instances.png
    │   ├── sg-bastion-sg.png
    │   ├── sg-dashboard-alb-sg.png
    │   ├── sg-dashboard-instance-sg.png
    │   ├── sg-counting-alb-sg.png
    │   ├── sg-counting-instance-sg.png
    │   ├── sg-dashboard-service.png
    │   ├── sg-counting-service.png
    │   ├── sg-dashboard-alb.png
    │   ├── sg-dashboard-target-group.png
    │   ├── sg-counting-alb.png
    │   ├── sg-counting-target-group.png
    │   └── sg-dashboard-test.png
    │
    ├── certificate/
    │   ├── tokyo-acm-certificate.png
    │   ├── singapore-acm-certificate.png
    │   └── https-listener.png
    │
    ├── route53/
    │   ├── route53-hosted-zone.png
    │   ├── namecheap-custom-dns.png
    │   ├── dns-delegation-test.png
    │   ├── japan-geolocation-record.png
    │   └── singapore-geolocation-record.png
    │
    └── testing/
        ├── dns-test.png
        ├── japan-vpn.png
        ├── dashboard-from-tokyo.png
        ├── singapore-vpn.png
        └── dashboard-from-singapore.png
```