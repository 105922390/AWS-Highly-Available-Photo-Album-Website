# Highly Available Photo Album Website on AWS

A highly available, scalable and secured AWS deployment of a photo album website, built as university coursework at Swinburne University of Technology. It extends an earlier single-server deployment into a multi-AZ architecture with a load balancer, auto scaling, private networking, least-privilege security groups, a network ACL, a restrictive S3 bucket policy and a Lambda function that creates thumbnails.

The full write-up, with console screenshots, is in [`docs/Assignment2_Highly_Available_Photo_Album_redacted.pdf`](docs/Assignment2_Highly_Available_Photo_Album_redacted.pdf). Account identifiers in the screenshots have been blacked out. This repository contains the report only, not the application source.

## Architecture

```
                         Internet
                            │ HTTP :80
                     ┌──────▼───────┐
                     │ Application  │  public subnets (us-east-1a, us-east-1b)
                     │ Load Balancer│  ELBSG: HTTP from the internet
                     └──────┬───────┘
                            │ HTTP :80 (WebServerSG: from the ELB only)
              ┌─────────────▼─────────────┐
              │ Auto Scaling Group (2–3)  │  private subnets, built from a custom AMI
              │ EC2 web servers           │  via a launch template
              └──────┬─────────────┬──────┘
        MySQL :3306  │             │ S3 API via NAT gateway
     (DBServerSG:    │             │
      web servers    │             ├──> S3 bucket ──triggers──> Lambda "CreateThumbnail"
      only)          │             │      (bucket policy)        (resizes uploaded PNGs)
              ┌──────▼──────┐
              │ RDS MySQL   │  private subnets
              └─────────────┘

Dev server (public subnet, DevServerSG): used to configure the app, create the AMI,
and manage the database through phpMyAdmin.
```

| Layer | What was built |
|---|---|
| Network | Custom VPC (`10.0.0.0/16`) across two AZs with a public and a private subnet in each, an internet gateway, a NAT gateway in the first public subnet, and separate public and private route tables |
| Compute | Dev server to build the app, custom AMI, launch template, Auto Scaling Group (min 2, max 3) with a target-tracking policy of 30 requests per target |
| Load balancing | Internet-facing Application Load Balancer with a target group and health checks across the private web servers |
| Storage | S3 bucket for uploaded and resized photos |
| Serverless | `CreateThumbnail` Lambda (Python, Pillow) that resizes images after upload |
| Database | RDS MySQL (`db.t4g.micro`) in the private subnets, with photo metadata managed through phpMyAdmin from the dev server |
| Access control | Four security groups, a network ACL on the private subnets, an IAM instance role, and an S3 bucket policy |

## Security design

**Security groups (chained, least privilege)**

| Group | Allows inbound |
|---|---|
| `ELBSG` | HTTP from the internet |
| `WebServerSG` | HTTP from the load balancer only |
| `DBServerSG` | MySQL (3306) from the web servers only |
| `DevServerSG` | Controls access to the dev server used for setup and database administration |

**Network ACL on the private subnets (`PrivateSubnetsNACL`)** adds a second, stateless layer: it allows HTTP, HTTPS and ephemeral return ports, allows MySQL from a specific private subnet range, denies ICMP between the dev server and the private web servers, and denies everything else by default. Ping tests confirmed the ICMP block.

**IAM:** an instance role attached through the launch template lets the web servers upload to S3 and invoke Lambda without any credentials stored on the instances.

**S3 bucket policy:** one statement grants the instance role access to the bucket, and a second allows `s3:GetObject` only when the request's `Referer` matches the load balancer's DNS name, so photos load through the website but not by direct link.

**Private-by-default:** the web servers and the database sit in private subnets with no public addresses. They reach the internet and S3 outbound through the NAT gateway.

## Testing

- Traffic through the load balancer was distributed across the web servers, and all targets reported healthy.
- Terminating an instance made the Auto Scaling Group launch a replacement automatically.
- Photos uploaded through the site were stored in S3 together with a resized copy, and their metadata appeared in RDS.
- ICMP between the dev server and the private web servers was blocked by the NACL, as designed.

## Challenges

- Getting the S3 bucket policy right, so the site could read images while direct access stayed blocked. A `Referer` condition on the load balancer's DNS name solved it.
- The Lambda function did not trigger or resize every image, which was a known behaviour in this lab and not a configuration fault.
- Managing a limited credit balance by stopping the dev server and the Auto Scaling Group instances after each test session.

## Limitations and what I would improve

- **The `Referer` condition is a soft control.** The header is set by the client and is easy to forge, so it only deters casual hotlinking. A stronger design keeps the bucket private and serves images through CloudFront with origin access control, or through pre-signed URLs.
- **No HTTPS.** The load balancer only listens on HTTP:80. I would add an ACM certificate, an HTTPS listener, and redirect HTTP to HTTPS.
- **Single NAT gateway.** If its AZ fails, the private subnets in the other AZ lose outbound access. One NAT gateway per AZ would remove that dependency.
- **Database availability.** The RDS instance is shown in a single AZ. Enabling Multi-AZ would complete the high-availability story.
- **Dev server exposure.** A public dev server running phpMyAdmin should be restricted to a known IP or replaced with Systems Manager Session Manager and an SSH tunnel.
- **Secrets.** Database credentials should live in AWS Secrets Manager, not in the application configuration.
- **Infrastructure as code.** Everything was built through the console. Rebuilding it in CloudFormation or Terraform would make it repeatable and reviewable.

## Repository layout

```
.
├── README.md
└── docs/
    └── Assignment2_Highly_Available_Photo_Album_redacted.pdf
```

## Services used

Amazon VPC, EC2, AMI, Auto Scaling, Elastic Load Balancing, NAT gateway, security groups and network ACLs, IAM, S3, Lambda, RDS (MySQL).
