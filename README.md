# EpicBook Application Deployment with Ansible

This repository contains Ansible automation for deploying the EpicBook application on AWS EC2 instances using Azure DevOps pipelines.

## Architecture

The deployment configures:

- **Application Server**: EpicBook application with systemd service
- **Web Server**: Nginx reverse proxy
- **Database**: RDS PostgreSQL connection
- **System Dependencies**: Required packages and configurations

## Prerequisites

- Azure DevOps account with pipeline access
- Self-hosted Azure DevOps agent (aws-ubuntu pool)
- SSH private key for EC2 access
- Running AWS infrastructure (EC2 instance + RDS database)

## Required Variable Groups

Configure these variable groups in Azure DevOps:

### terraform-variables

- `TF_RDS_ADMIN`: Database admin username
- `TF_RDS_ADMIN_PASSWORD`: Database admin password

### database-vars

- `db_name`: Application database name
- `db_user`: Application database user
- `db_password`: Application database password

## Secure Files

Upload to Azure DevOps Library:

- `id_rsa`: SSH private key for EC2 access

## Pipeline Parameters

When running the pipeline, provide:

- **VM Public IP**: EC2 instance public IP address
- **RDS Endpoint**: Database connection endpoint

## Deployment Process

### Manual Trigger

1. Navigate to Azure DevOps Pipelines
2. Select the EpicBook deployment pipeline
3. Click "Run pipeline"
4. Enter required parameters:
   - VM Public IP (from infrastructure outputs)
   - RDS Endpoint (from infrastructure outputs)
5. Run the pipeline

### Pipeline Stages

#### Parameter Validation

- Validates required parameters are provided
- Fails early if inputs are missing

#### SSH Setup

- Downloads private key from secure files
- Configures SSH with proper permissions
- Sets up host key checking bypass

#### Dynamic Configuration

- Creates `inventory.ini` with target server IP
- Generates `vars/db.yml` with database configuration
- Uses parameter values and variable groups

#### Ansible Deployment

- Runs the main playbook: `epicbook.yaml`
- Deploys application across multiple roles
- Configures services and dependencies

## Ansible Structure

### Playbook

- `epicbook.yaml`: Main deployment playbook

### Roles

- **app**: System dependencies and base configuration
- **rds**: Database connectivity setup
- **epicbook**: Application deployment and service configuration
- **nginx**: Web server setup and reverse proxy

### Variables

- `vars/env.yml`: Environment-specific settings
- `vars/db.yml`: Database configuration (generated dynamically)

### Inventory

- `inventory.ini`: Target servers (generated dynamically)

## Configuration Files

### Application Service

- Template: `roles/epicbook/templates/epicbook.service.j2`
- Creates systemd service for the application

### Application Config

- Template: `roles/epicbook/templates/config.json.j2`
- Database and application settings

### Nginx Configuration

- Template: `roles/nginx/templates/nginx.conf.j2`
- Reverse proxy setup

## Deployment Flow

1. **Infrastructure Setup** (separate pipeline)
   - Deploy EC2 instance and RDS database
   - Get Public IP and RDS endpoint

2. **Application Deployment** (this pipeline)
   - Input infrastructure details as parameters
   - Configure SSH access
   - Deploy application stack
   - Start services

## Security Notes

- SSH keys managed through Azure DevOps secure files
- Database credentials stored in variable groups
- Host key checking disabled for automation
- Private subnets used for database access

## Troubleshooting

### Common Issues

- **SSH Connection Failed**: Verify private key matches EC2 key pair
- **Database Connection**: Check RDS endpoint and security groups
- **Service Start Failed**: Review application logs and configuration

### Verification Commands

```bash
# Check application service
sudo systemctl status epicbook

# Check nginx status
sudo systemctl status nginx

# View application logs
sudo journalctl -u epicbook -f

# Test database connection
psql -h <rds-endpoint> -U <username> -d <database>
```

## Manual Deployment

For local testing, you can run Ansible directly:

```bash
# Install dependencies
ansible-galaxy install -r requirements.yml

# Run playbook
ansible-playbook -i inventory.ini epicbook.yaml
```

## Integration

This deployment pipeline works with the infrastructure pipeline:

1. Run infrastructure pipeline to create AWS resources
2. Copy outputs (Public IP, RDS Endpoint)
3. Run this deployment pipeline with the infrastructure outputs
4. Application will be deployed and accessible via the public IP
