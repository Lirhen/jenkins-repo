# Jenkins Platform Repository
Docker-based Jenkins setup for CI/CD pipeline on EC2

## Overview
This repository contains the Docker setup for running Jenkins on EC2 instance as part of a CI/CD pipeline. The Jenkins instance is configured to build Docker images, run tests, push to ECR, and deploy to production servers.

## Repository Structure
```
jenkins-platform-repo/
├── Dockerfile                 # Jenkins image with Docker CLI and AWS CLI
├── docker-compose.yml         # Docker Compose configuration  
├── plugins.txt               # Jenkins plugins list
└── README.md                 # This file
```

## Prerequisites

### EC2 Instance Requirements
- **Instance Type**: t3.medium or larger (Jenkins requires adequate memory)
- **Operating System**: Amazon Linux 2
- **Security Group**: Allow inbound traffic on ports:
  - 22 (SSH)
  - 8080 (Jenkins Web UI)
- **Storage**: Minimum 20GB EBS volume

### Network Configuration
Ensure Security Group allows:
```
Type         Protocol  Port    Source
SSH          TCP       22      Your IP/VPC CIDR  
HTTP         TCP       8080    0.0.0.0/0 (or restricted IPs)
```

## Installation Steps

### 1. Prepare EC2 Instance

#### Connect to EC2 Instance
```bash
ssh -i your-key.pem ec2-user@your-jenkins-server-ip
```

#### Install Docker
```bash
# Update system packages
sudo yum update -y

# Install Docker
sudo yum install -y docker

# Start and enable Docker service
sudo systemctl start docker
sudo systemctl enable docker

# Add ec2-user to docker group (allows running docker without sudo)
sudo usermod -aG docker ec2-user

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Verify installations
docker --version
docker-compose --version
```

#### Important: Re-login after Docker group addition
```bash
exit
ssh -i your-key.pem ec2-user@your-jenkins-server-ip
```

### 2. Deploy Jenkins Platform

#### Clone Platform Repository
```bash
git clone https://github.com/your-username/jenkins-platform-repo.git
cd jenkins-platform-repo
```

#### Verify Docker Socket Permissions
```bash
# Check Docker socket group ownership
ls -la /var/run/docker.sock

# Get Docker group ID (usually 999)
getent group docker
```

#### Start Jenkins
```bash
# Build and start Jenkins container
docker-compose up -d

# Verify container is running
docker ps

# Check Jenkins logs
docker logs jenkins
```

### 3. Initial Jenkins Setup

#### Get Initial Admin Password
```bash
# Extract initial admin password from Jenkins logs
docker logs jenkins 2>&1 | grep -A 5 "Please use the following password"

# Or read directly from container
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

#### Access Jenkins Web Interface
1. Open browser: `http://your-ec2-public-ip:8080`
2. Enter the initial admin password
3. Select "Install suggested plugins"
4. Create first admin user
5. Configure Jenkins URL (use your EC2 public IP)

### 4. Configure Jenkins for CI/CD

#### Install Additional Plugins
Navigate to **Manage Jenkins > Manage Plugins > Available** and install:
- Docker Pipeline
- SSH Agent Plugin  
- AWS Credentials Plugin (if not using IAM roles)
- Multibranch Scan Webhook Trigger
- Generic Webhook Trigger

#### Configure System Settings
**Manage Jenkins > Configure System:**
- **Jenkins URL**: `http://your-ec2-public-ip:8080`
- **GitHub Server**: Add GitHub.com (if needed)
- **Global Tool Configuration**: Configure Git (usually auto-detected)

### 5. Docker Integration Troubleshooting

If you encounter Docker permission errors in pipelines:

#### Quick Fix (Temporary)
```bash
# Fix Docker socket permissions
docker exec -u root jenkins chmod 666 /var/run/docker.sock
```

#### Permanent Fix
The `docker-compose.yml` is configured with:
```yaml
user: "0:999"          # Root user with docker group
group_add:
  - "999"              # Docker group ID
```

If issues persist, verify Docker group ID:
```bash
# Check actual Docker group ID on your system
stat -c '%g' /var/run/docker.sock

# Update docker-compose.yml group_add value if different from 999
```

## File Descriptions

### Dockerfile
Builds Jenkins image with:
- **Base**: jenkins/jenkins:lts-jdk17
- **Docker CLI**: For building/pushing images
- **AWS CLI**: For ECR authentication
- **Plugins**: Pre-installed from plugins.txt
- **User Configuration**: Jenkins user added to docker group

### docker-compose.yml
- **Ports**: Maps 8080 (web) and 50000 (agent communication)
- **Volumes**: Persists Jenkins data and provides Docker socket access
- **Environment**: Docker integration settings
- **User/Group**: Configured for Docker access

### plugins.txt
Essential plugins for CI/CD pipeline:
- `workflow-aggregator`: Pipeline support
- `docker-workflow`: Docker agents in pipelines
- `git`: Git repository integration
- `ssh-agent`: SSH credential management
- `aws-credentials`: AWS integration
- Additional plugins for GitHub webhooks and security

## Maintenance

### Backup Jenkins Data
```bash
# Create backup of Jenkins data
docker run --rm -v jenkins-data:/data -v $(pwd):/backup alpine tar czf /backup/jenkins-backup-$(date +%Y%m%d).tar.gz -C /data .
```

### Update Jenkins
```bash
# Pull latest image and recreate container
docker-compose down
docker-compose pull
docker-compose up -d
```

### View Logs
```bash
# Real-time logs
docker logs -f jenkins

# Recent logs
docker logs --tail 100 jenkins
```

### Restart Jenkins
```bash
# Restart container
docker-compose restart

# Or restart Jenkins service (without container restart)
docker exec jenkins java -jar /var/jenkins_home/war/WEB-INF/jenkins-cli.jar -s http://localhost:8080 restart
```

## Security Considerations

### Network Security
- Restrict port 8080 access to required IPs only
- Use HTTPS in production (configure reverse proxy)
- Keep Jenkins and plugins updated

### Jenkins Security Settings
**Manage Jenkins > Configure Global Security:**
- **Enable security**: Always enabled
- **Security Realm**: Jenkins' own user database
- **Authorization**: Logged-in users can do anything (or configure matrix-based security)
- **CSRF Protection**: Enable with default crumb issuer

### Secrets Management
- Store sensitive data in Jenkins Credentials
- Use IAM roles instead of hardcoded AWS keys when possible
- Never commit secrets to Git

## Troubleshooting

### Common Issues

#### Container Won't Start
```bash
# Check Docker daemon status
sudo systemctl status docker

# Check disk space
df -h

# Check container logs
docker logs jenkins
```

#### Can't Access Jenkins Web Interface
```bash
# Verify container is running and ports are mapped
docker ps

# Check Security Group allows port 8080
# Test local access
curl http://localhost:8080

# Check if another service is using port 8080
sudo netstat -tlnp | grep 8080
```

#### Docker Permission Errors in Pipelines
```bash
# Check Docker socket ownership
ls -la /var/run/docker.sock

# Verify Jenkins container has docker group access
docker exec jenkins groups jenkins

# Fix permissions if needed
docker exec -u root jenkins usermod -aG docker jenkins
docker-compose restart
```

#### Out of Disk Space
```bash
# Check disk usage
df -h

# Clean up Docker resources
docker system prune -a

# Remove old Jenkins builds (via web interface)
# Manage Jenkins > Configure System > Discard Old Builds
```

### Performance Optimization

#### For Low-Memory Instances
Add to `docker-compose.yml`:
```yaml
environment:
  - JAVA_OPTS=-Xmx512m -Xms256m
```

#### For High-Load Scenarios
- Increase instance size to t3.large or larger
- Add more executors: **Manage Jenkins > Configure System > # of executors**
- Configure build history retention to prevent disk fill-up

## Integration with Application Repository

### Expected Workflow
1. **Application repository** contains:
   - Jenkinsfile (pipeline definition)
   - Dockerfile (application containerization)
   - Source code and tests
   
2. **This Jenkins platform**:
   - Provides runtime environment for pipelines
   - Builds Docker images
   - Pushes to ECR
   - Deploys to production servers

### Required Jenkins Credentials
For the CI/CD pipeline to work, configure these credentials:
- **SSH Key** (`production-ssh-key`): Access to production servers
- **AWS Credentials** (`aws-ecr-credentials`): ECR push/pull (if not using IAM roles)

## Support

### Log Locations
- **Jenkins logs**: `docker logs jenkins`  
- **Pipeline logs**: Available in Jenkins web interface
- **System logs**: `/var/log/messages` on EC2 instance

### Useful Commands
```bash
# Enter Jenkins container shell
docker exec -it jenkins bash

# Check Java version in container  
docker exec jenkins java -version

# Check installed plugins
docker exec jenkins java -jar /var/jenkins_home/war/WEB-INF/jenkins-cli.jar -s http://localhost:8080 list-plugins
```

This Jenkins platform provides a robust foundation for CI/CD pipelines with proper Docker integration and security configurations.
