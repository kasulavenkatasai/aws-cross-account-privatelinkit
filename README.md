# AWS Cross-Account PrivateLink with MariaDB

## Overview

This project demonstrates how to establish secure private connectivity between two separate AWS accounts using **AWS PrivateLink**. An EC2 instance in the consumer account connects to a MariaDB database hosted in the provider account through an internal Network Load Balancer (NLB), without exposing the database to the public internet.

The infrastructure is provisioned using AWS CloudFormation templates.

## Architecture

```text
              AWS ACCOUNT B — CONSUMER
              VPC: 172.31.0.0/16
              ┌──────────────────────────┐
              │  Private EC2 Instance    │
              │  Amazon Linux 2023       │
              └────────────┬─────────────┘
                           │
                           │ TCP 3306
                           ▼
              ┌──────────────────────────┐
              │  Interface VPC Endpoint  │
              └────────────┬─────────────┘
                           │
═══════════════════════════╪══════════════════════════
                  AWS PrivateLink
═══════════════════════════╪══════════════════════════
                           │
                           ▼
              AWS ACCOUNT A — PROVIDER
              VPC: 10.0.0.0/16
              ┌──────────────────────────┐
              │  VPC Endpoint Service    │
              └────────────┬─────────────┘
                           │
                           ▼
              ┌──────────────────────────┐
              │  Internal Network Load   │
              │  Balancer (NLB)          │
              └────────────┬─────────────┘
                           │
                           ▼
              ┌──────────────────────────┐
              │  Private EC2 Instance    │
              │  MariaDB Server          │
              └────────────┬─────────────┘
                           │
                           ▼
              Database: privatelink_demo
```

## AWS Services Used

- **Amazon VPC:** Isolated networking for the provider and consumer accounts.
- **AWS PrivateLink:** Private connectivity between the two AWS accounts.
- **Network Load Balancer:** Routes TCP traffic to the provider database.
- **Amazon EC2:** Hosts MariaDB and provides a consumer-side test instance.
- **AWS CloudFormation:** Automates infrastructure deployment.
- **AWS Systems Manager Session Manager:** Provides private instance access without requiring SSH or a public IP address.
- **AWS IAM:** Controls instance permissions and service access.
- **AWS Secrets Manager:** Stores the generated MariaDB application credentials.
- **Amazon VPC Endpoints:** Provides private access to required AWS services, including Systems Manager and S3-hosted package repositories.

## Repository Structure

```text
aws-cross-account-privatelinkit/
├── README.md
├── account-a-provider/
│   └── mysqlprovider.yaml
└── account-b-consumer/
    └── mysqlconsumer.yaml
```

## Network Configuration

| Component | Configuration |
|---|---|
| AWS Region | US East (N. Virginia) — `us-east-1` |
| Provider VPC | `10.0.0.0/16` |
| Consumer VPC | `172.31.0.0/16` |
| Provider subnets | `10.0.1.0/24`, `10.0.2.0/24` |
| Consumer subnets | `172.31.10.0/24`, `172.31.11.0/24` |
| Database engine | MariaDB 10.5 |
| Database name | `privatelink_demo` |
| Application username | `privatelink_app` |
| Database port | TCP 3306 |
| Connectivity | AWS PrivateLink |

## Deployment Steps

### 1. Deploy the Provider Infrastructure

Sign in to **AWS Account A** and open CloudFormation in `us-east-1`.

1. Choose **Create stack → With new resources (standard)**.
2. Upload `account-a-provider/mysqlprovider.yaml`.
3. Configure the template parameters.
4. Review the configuration and create the stack.
5. Wait for the stack to reach `CREATE_COMPLETE`.
6. Open the stack's **Outputs** tab and copy `ProviderEndpointServiceName`.

The provider stack provisions the VPC, private subnets, database EC2 instance, internal NLB, target group, endpoint service, security groups, IAM permissions, and required service endpoints.

### 2. Deploy the Consumer Infrastructure

Sign in to **AWS Account B**.

1. Open CloudFormation in `us-east-1`.
2. Create a stack using `account-b-consumer/mysqlconsumer.yaml`.
3. Set `ProviderEndpointServiceName` to the value copied from Account A.
4. Review the parameters and create the stack.
5. Wait for `CREATE_COMPLETE`.

The consumer stack creates the consumer VPC, private EC2 instance, Systems Manager endpoints, MariaDB interface endpoint, and S3 Gateway Endpoint.

### 3. Accept the PrivateLink Connection

Return to Account A.

1. Open **VPC → Endpoint services**.
2. Select the provider endpoint service.
3. Open the endpoint connections section.
4. Accept the connection request from Account B if acceptance is required.
5. Verify that the consumer endpoint becomes `Available`.

### 4. Connect to the Consumer Instance

In Account B:

1. Open **EC2 → Instances**.
2. Select the consumer EC2 instance.
3. Choose **Connect → Session Manager → Connect**.
4. Verify that the terminal session opens successfully.

### 5. Install the MariaDB Client

The consumer instance uses a private subnet. An S3 Gateway Endpoint is configured to provide private access to S3-hosted Amazon Linux package repositories.

Install the MariaDB client:

```bash
sudo dnf install mariadb105 -y
```

### 6. Test PrivateLink Connectivity

Resolve the consumer interface endpoint DNS name:

```bash
nslookup <consumer-endpoint-dns-name>
```

Test TCP connectivity on port 3306:

```bash
timeout 5 bash -c '</dev/tcp/<consumer-endpoint-dns-name>/3306' \
  && echo "TCP connection successful" \
  || echo "TCP connection failed"
```

Replace `<consumer-endpoint-dns-name>` with the DNS name displayed in the consumer VPC endpoint details.

### 7. Connect to MariaDB

Retrieve the database password from the provider's Secrets Manager secret. Do not commit the password to GitHub.

Run the following command from the consumer EC2 instance:

```bash
mariadb -h <consumer-endpoint-dns-name> \
  -P 3306 \
  -u privatelink_app \
  -p privatelink_demo
```

Enter the password when prompted.

### 8. Verify Database Access

After connecting, execute:

```sql
SHOW TABLES;
```

Verify the sample record:

```sql
SELECT * FROM connection_test;
```

Expected table:

| id | message |
|---:|---|
| 1 | PrivateLink database initialized |

The database connection and sample SQL query were successfully verified in this project.

## Validation Results

The following tests were completed successfully:

- [x] Provider CloudFormation stack deployed.
- [x] Consumer CloudFormation stack deployed.
- [x] Cross-account endpoint connection accepted.
- [x] Consumer interface endpoint reached `Available`.
- [x] Endpoint DNS resolved to private IP addresses.
- [x] TCP connectivity to port 3306 succeeded.
- [x] MariaDB client installed on the consumer EC2 instance.
- [x] Database authentication succeeded.
- [x] `connection_test` table was retrieved.
- [x] Sample database record was queried successfully.

## Security Considerations

- The database is accessed through AWS PrivateLink rather than a public database endpoint.
- The provider uses an internal Network Load Balancer.
- Security groups restrict database access to the intended network path.
- Database credentials are stored in AWS Secrets Manager.
- EC2 access uses Systems Manager Session Manager.
- Never commit AWS access keys, database passwords, secret values, or other sensitive information to the repository.
- Review IAM permissions, security group rules, and endpoint policies before using this architecture in production.

## Cost Considerations

This project can incur AWS charges even when deployed in private subnets.

Potentially chargeable resources include:

- EC2 instances
- Network Load Balancer
- Interface VPC endpoints
- AWS Secrets Manager
- Data processing and related service usage

An S3 Gateway Endpoint has no additional endpoint charge. However, ordinary S3 data transfer or other related charges may still apply.

Review AWS Billing before deployment, and delete the lab resources when they are no longer needed.

## Learning Outcomes

This project demonstrates practical experience with:

- Cross-account AWS networking
- AWS PrivateLink and VPC endpoint services
- Network Load Balancer configuration
- VPC subnet and route table design
- Security groups and IAM roles
- AWS Systems Manager Session Manager
- AWS Secrets Manager
- Infrastructure as Code using CloudFormation
- MariaDB connectivity and SQL validation
- Troubleshooting DNS, TCP connectivity, package repository access, and database authentication

## Author

**Kasula Venkata Sai**

GitHub: [@kasulavenkatasai](https://github.com/kasulavenkatasai)

Repository: [aws-cross-account-privatelinkit](https://github.com/kasulavenkatasai/aws-cross-account-privatelinkit)
