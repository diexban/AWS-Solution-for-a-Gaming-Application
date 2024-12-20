# AWS Solution For A Web Application

## Infrastructure Design Proposal for AWS Web Application Hosting

As part of the infrastructure design for this new web application, I am proposing **two solutions** for hosting the application on AWS. Since I don't know whether the application will be built using container images, the two solutions address both possibilities:

1. **Using AWS Fargate**  
   _(For container-based workloads)_  

2. **Using EC2 Instances**  
   _(For traditional virtual machines)_

## Solution #1

<img width="1310" alt="396089984-babedf79-e1fc-4622-93a0-9339c00334ae" src="https://github.com/user-attachments/assets/244b741d-9ddc-4f40-865b-09ed176be16b" />


## Explanation and Components Used in the Architecture Diagram

For my first solution, I decided to create a VPC within a region and distribute components across two Availability Zones to ensure high availability and fault tolerance. I envisioned the services being built with a containerization mindset so they are provided as images, so I created two ECS clusters: one for the web layer and one for the application layer.

In each Availability Zone, I deployed Fargate instances running both front-end and back-end services using images pulled from what the developers push to Amazon ECR. These instances are separated into their own clusters and subnets to ensure they have the necessary resources and network ranges to scale based on traffic and computing load.

To handle load distribution between the web and app layers, I set up an internal load balancer with a security group that balances traffic between the web layer and the app layer. Each Fargate instance is placed inside its own security group, allowing communication only between layers and the necessary services. This separation minimizes the risk of security breaches.

For the database layer, I have set up an RDS instance with synchronous replication in the second Availability Zone, which ensures failover capabilities in case of an issue. This database is also placed inside its own security group for added security. I opted for a single RDS instance, Additionally, I added two ElastiCache Redis node to offload some of the load from the database using caching. These nodes have a separate Security Group as well

For content distribution, I use CloudFront to accelerate delivery and cache static content. CloudFront is connected to an Application Load Balancer in a public subnet of my VPC and it's own security group, which balances traffic to the web layer. For storage, I leverage S3 to serve static content, store backups, and store logs.

In terms of monitoring, I configured CloudWatch to monitor the health of each component and CloudTrail to track any significant changes made to the environment, helping with auditing and troubleshooting.


### 1. Networking Layer
- **AWS Region**  
- **Virtual Private Cloud (VPC)**
- **Public Subnet** (used for Application Load Balancer)
- **Private Subnets**
- **AWS Availability Zone 1**
- **AWS Availability Zone 2**
- **Security Groups**

### 2. Compute Layer
- **Amazon ECS Cluster in the web and app layer**
- **AWS Fargate** (used in both Web Layer and App Layer)

### 3. Load Balancing
- **Application Load Balancer** (Public-facing)
- **Internal Load Balancer**

### 4. Database Layer
- **Amazon RDS** (Relational Database Service)
- **RDS Synchronous Replica** (for high availability and failover)
- **ElastiCache Redis Nodes** (used as caching layers in both Availability Zones)

### 5. Storage Layer
- **Amazon S3** (for log, backup and static content storage)

### 6. Content Delivery
- **Amazon CloudFront** (used as a CDN for caching and accelerating content)

### 7. Monitoring and Security
- **AWS CloudWatch** (for monitoring and logging)
- **AWS CloudTrail** (for auditing )
- **AWS IAM** (Identity and Access Management for security control)

### 8. Deployment & Container Registry
- **Amazon Elastic Container Registry (ECR)** (for container image storage)

##  Discussion

■ How does each choice contribute to cost-effectiveness against high
availability?
- A key point in my first solution that contributes to cost-effectiveness is the use of Fargate. As a managed service, Fargate automatically scales based on demand, ensuring that the system performs well under varying loads without the need to over-provision resources. This approach is more cost-effective than using EC2 instances, where over-provisioning could lead to unnecessary expenses.

- Another important decision I made was to use RDS with synchronous replication. While having a replica in a second availability zone is cheaper than using two separate RDS instances for high availability, opting for a single RDS instance without a replica could save costs further. However, this would compromise high availability and fault tolerance. The additional cost of a replica I believe is justified to reduce the risk of downtime, which could lead to greater losses.

- I also opted on having two separete clusters so we can optimize resource allocation for each layer meaning if the web layer needs to scale independently of the app layer we would be able to do this. This could introduce indirect costs due to the increased complexity but in the long run it's better to have the layers as independent clusters

- Lastly while using CloudFront adds some cost, it provides significant performance benefits by reducing strain on the web layer. With caching in place, the web layer does not need to scale as frequently, keeping costs lower overall. Similarly, using Redis adds costs for the caching layer but significantly reduces load on the database, helping maintain performance at scale.

■ What trade-offs, if any, did you make between cost and
performance?

- I opted for synchronous replication in RDS to ensure high availability, even though this incurs additional cost. The alternative of using a single-instance RDS would reduce costs but compromise the application's resilience, which could lead to greater losses in case of downtime.

- Using CloudFront and Redis incurs some additional cost, but the trade-off is worth it for the significant improvements in performance and scalability. This reduces the load on the web layer and the database, ensuring that both components are more efficient and perform well under varying traffic.

■ Discuss any cost implications of your scaling policies and how they
adapt to varying traffic loads.

While I can't really provide precise cost without knowing the exact traffic patterns, I would implement the following scaling policies to keep costs under control:

- A Target Tracking Scaling: This policy will automatically adjust the number of tasks for computing instances based on usage. By scaling the application dynamically according to traffic, we can avoid over-provisioning, which helps keep costs low.

- A Log Retention Policy in CloudWatch: To further reduce costs, I would apply a short log retention policy in CloudWatch. Logs would be transferred to S3 with its own retention policy, ensuring that unnecessary data is not stored for long periods, reducing storage costs.

- Reserved Instances for RDS: To manage database costs effectively while ensuring high availability, I would purchase Reserved Instances for RDS as possible. This would give us predictable pricing and lower costs over time, while still allowing us to handle variable traffic loads.

- For S3 buckets I would apply a retention policy to keep cost low and change the storage tier to the one really needed
   
## Solution #2
![AWS Solution Fargate (3)](https://github.com/user-attachments/assets/c472edbd-690a-436c-9441-d91506746eae)

## Explanation and Components Used in Solution #2

For my second solution, I implemented a more traditional 3-tier architecture using EC2 instances as the core compute layer. Rather than detailing all the components in the design, I'll focus on the key differences compared to the original setup:

- Instead of using AWS Fargate to run the application, I opted for EC2 instances.
These instances are placed inside an Auto Scaling Group (ASG) to ensure scalability. The ASG automatically adjusts the number of instances based on traffic or application demands, adding new instances during peak times and reducing them during low-traffic periods.
This approach gives greater control over the compute layer, especially if the application requires custom configurations or additional resources that are easier to manage with EC2 instances.


- To streamline database management and benefit from automatic scaling, I replaced the Amazon RDS instance with Amazon Aurora.
Aurora provides features such as automatic replication, high availability, and dynamic scaling, which can handle varying workloads more efficiently than traditional RDS.
This change simplifies database maintenance but also reduces costs by scaling storage and compute independently as needed.

 - To reduce costs, I removed Amazon CloudFront as a content distribution layer.
While it provides significant performance and security benefits by caching content at edge locations, the architecture now routes requests directly to the load balancer. This approach simplifies the architecture and avoids the additional costs associated with maintaining a CDN.


## Extra Disaster Recovery Stragedy
<img width="581" alt="image" src="https://github.com/user-attachments/assets/c7a46929-82d3-44ed-8fdf-8c032bf1e9e9" />

As an extra to task #1 and beacuse we don't know how crucial uptime is with the new app I would recommed to have a region failover plan, while a region failure is very rare, it’s still important to have a solid disaster recovery plan. I’d go with a Pilot Light approach, which works well with either of my solutions. This means replicating the infrastructure in another AWS region but keeping it running with minimal resources—just enough to stay functional. If something goes wrong in the primary region, I can quickly fail over to the backup region, spin up the needed resources, and get everything back online. Once the issue is resolved, I can clean up any unnecessary resources. It’s a cost-effective way to stay prepared without running a full-scale duplicate environment all the time.

Image reference: [AWS Blog](https://aws.amazon.com/blogs/architecture/disaster-recovery-dr-architecture-on-aws-part-iii-pilot-light-and-warm-standby/).

## Task #2
For task number two, I would convert my structure into Terraform code and store it in a GitHub repository. I could create a variables.tf file with environment-specific and client-specific .tfvars files so we can reuse the main.tf file in different situations. This would solve any reusability issues that might arise. I would also structure my project with modules to avoid any unnecessary duplication.

## Infrastructure as Code with Terraform

The following are two snippets of how I would approach the first steps of turning the design into IaC.

variables.tf
```hcl

variable "vpc_cidr_block" {
  description = "CIDR block"
  type        = string
  default     = 
}


variable "environment" {
  description = "The deployment environment)"
  type        = string
}


variable "ecs_cluster_name" {
  description = "ECS cluster"
  type        = string
}


variable "task_count" {
  description = "The number of tasks to run in the ECS service"
  type        = number
  default     = 2
}


variable "container_image" {
  description = "The container image"
  type        = string
}
```

main.tf
```hcl
### VPC
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr_block

  tags = {
    Name = "VPC-${var.environment}"
  }
}

#### ECS Cluster
resource "aws_ecs_cluster" "ecs_cluster" {
  name = var.ecs_cluster_name
}

### ECS Fargate Service
resource "aws_ecs_service" "app_service" {
  name            = "app-service-${var.environment}"
  cluster         = aws_ecs_cluster.ecs_cluster.id
  launch_type     = "FARGATE"
  desired_count   = var.task_count

  task_definition = aws_ecs_task_definition.app_task.arn
}

### ECS Task 
resource "aws_ecs_task_definition" "app_task" {
  family                   = "app-task-${var.environment}"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "512"
  memory                   = "1024"

  container_definitions = jsonencode([
    {
      name  = "app-container"
      image = var.container_image
      portMappings = [
        {
          containerPort = 6020
          hostPort      = 2060
        }
      ]
    }
  ])
}
```
For the pipeline I would create a job in Jenkins where on commit it would trigger all checks and steps until Terraform Apply and would trigger a status to slack or email where we can check the status of the deployment and the logs

![Untitled Diagram (1)](https://github.com/user-attachments/assets/780b9ef3-3db4-4421-b9ad-add9d6dc13a4)

### Git Commit:

developer pushes changes to a repository containing Terraform code 
This commit acts as the trigger for the pipeline.

 ### Jenkins Job:

- terraform init: Initialize Terraform working directory.
- terraform validate: Validate the syntax and structure of Terraform code.
- terraform plan: Show the resources that will be created, changed, or destroyed.
- terraform apply: Apply the changes to provision the infrastructure.
- Terraform Apply Successful:

If the terraform apply step completes successfully then resources are created or modified in AWS.

### Slack or Email Notification:

Notifications are sent to inform about the success of the Terraform job.
