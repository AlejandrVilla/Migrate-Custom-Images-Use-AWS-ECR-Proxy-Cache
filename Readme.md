# AWS ECR Migration & Proxy Cache Setup

## 🚀 Overview
This project migrates custom images to AWS ECR and sets up AWS ECR Pull-Through Cache for public images.

## 🎯 Objectives
✅ Migrate custom images to AWS ECR  
✅ Use AWS ECR Proxy Cache for base images  
✅ Disable Docker Hub to enforce security  

## Set Up
1. Install aws-cli
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

2. Configure aws cli
```
aws configure
# add your access key and secret access key
```

### useful-links
- [aws cli](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [docker engine](https://docs.docker.com/engine/install/ubuntu/)

## 🛠️ Setup Steps
1. Create AWS ECR repositories  
```
aws ecr create-repository --repository-name [REPOSITORY_NAME] --region us-east-1
```

2. Configure AWS ECR Pull-Through Cache  
- Crear un secret con las credenciales de docker hub y guardar el SECRET_ARN
```
aws ecr create-pull-through-cache-rule \
    --ecr-repository-prefix [REPOSITORY_NAME] \
    --upstream-registry-url registry-1.docker.io \
    --credential-arn [SECRET_ARN] \
    --region us-east-1
```
- Verify the cache rule exists
```
aws ecr describe-pull-through-cache-rules --region us-east-1
```

3. Migrate base images via AWS ECR Proxy  
- Make sure you are authenticated to Amazon ECR
```
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin [COUNT_ID].dkr.ecr.us-east-1.amazonaws.com
```
```
docker pull [COUNT_ID].dkr.ecr.us-east-1.amazonaws.com/[REPOSITORY_NAME]/[IMAGE_NAME]
```
- Verify images are stored in AWS ECR
```
aws ecr list-images --repository-name [REPOSITORY_NAME] --region us-east-1
```

4. Build and push custom images  
```
docker build -t [IMAGE_NAME] -f Dockerfile.my-app .
# or pull an image
docker pull [IMAGE_NAME]
```
```
docker tag [IMAGE_NAME] [COUNT_ID].dkr.ecr.us-east-1.amazonaws.com/[RESPOSITORY_NAME]
```

5. Disable Docker Hub access at the Network Level (Use a firewall rule to block access to Docker Hub, for UFW):
- Enable UFW
```
sudo ufw enable
```
- Find Docker Hub's IP addresses
```
nslookup registry-1.docker.io
```

- Example output

```
Non-authoritative answer:
Name:   registry-1.docker.io
Address: 44.208.254.194
Name:   registry-1.docker.io
Address: 98.85.153.80
Name:   registry-1.docker.io
Address: 3.94.224.37
Name:   registry-1.docker.io
Address: 2600:1f18:2148:bc00:5cac:48a0:7f88:7266
Name:   registry-1.docker.io
Address: 2600:1f18:2148:bc02:22:27bd:19a8:870c
Name:   registry-1.docker.io
Address: 2600:1f18:2148:bc01:f43d:e203:cafd:8307
```

- Block Docker Hub's IP Using UFW
```
sudo ufw deny out to 44.208.254.194 port 443 proto tcp
sudo ufw deny out to 98.85.153.80 port 443 proto tcp
sudo ufw deny out to 3.94.224.37 port 443 proto tcp
sudo ufw deny out to 2600:1f18:2148:bc00:5cac:48a0:7f88:7266 port 443 proto tcp
sudo ufw deny out to 2600:1f18:2148:bc02:22:27bd:19a8:870c port 443 proto tcp
sudo ufw deny out to 2600:1f18:2148:bc01:f43d:e203:cafd:8307 port 443 proto tcp
```

- Reload UFW to apply changes
```
sudo UFW reload
```

- Verify UFW rules
```
sudo ufw status numbered
```

- Try pulling an image
```
docker pull alpine
```

## 📌 Notes
- AWS ECR will now serve as the single source for all container images.
- Docker Hub is **disabled** to prevent unauthorized pulls.