# ShopiTry: Enterprise AWS Production Deployment Guide

This document provides a comprehensive, step-by-step production deployment guide for the **ShopiTry** full-stack microservices ecosystem. It covers provisioning databases on MongoDB Atlas, deploying serverless functions to AWS Lambda, deploying core microservices to AWS EC2 / VM Instances, and hosting twin React SPAs on AWS S3 with CloudFront CDN distributions.

---

## 🔑 Production Environment & Database Parameters

| Parameter | Value |
| :--- | :--- |
| **Application Name** | `ShopiTry` |
| **MongoDB Atlas Cluster Name** | `edublitz` |
| **Database Username** | `linux` |
| **Database Password** | `redhat` |
| **Connection URI Pattern** | `mongodb+srv://<db_username>:<db_password>@edublitz.laegfsa.mongodb.net/?appName=edublitz` |
| **Catalog Database Name** | `catalog_db` |
| **Cart Database Name** | `cart_db` |
| **Orders Database Name** | `orders_db` |

---

## 📋 Architecture & Deployment Topology

| Component | Target Infrastructure | Deployment Primitive | Target URL / Port |
| :--- | :--- | :--- | :--- |
| **`storefront`** | AWS S3 Bucket + CloudFront CDN | React Production SPA (`dist/`) | `https://store.shopitry.com` |
| **`admin-dashboard`** | AWS S3 Bucket + CloudFront CDN | React Production SPA (`dist/`) | `https://admin.shopitry.com` |
| **`gateway-service`** | AWS EC2 (VM Server 1) | Node.js + PM2 + Nginx Proxy | `https://api.shopitry.com` (5000) |
| **`catalog-service`** | AWS EC2 (VM Server 2) | Node.js + PM2 + MongoDB Atlas | Private VPC (5001) |
| **`cart-service`** | AWS EC2 (VM Server 3) | Node.js + PM2 + MongoDB Atlas | Private VPC (5002) |
| **`order-service`** | AWS EC2 (VM Server 4) | Node.js + PM2 + MongoDB Atlas | Private VPC (5003) |
| **`payment-service`** | AWS Lambda 1 | Node.js 20.x Runtime | AWS Lambda API Gateway (5004) |
| **`notification-service`** | AWS Lambda 2 | Node.js 20.x Runtime | AWS Lambda API Gateway (5005) |

---
```
url to install mongosh - https://www.mongodb.com/docs/mongodb-shell/install/?operating-system=linux&linux-distribution=ubuntu&ubuntu-version=noble 
url to install node npm - https://www.digitalocean.com/community/tutorials/how-to-install-node-js-on-ubuntu-22-04
```
## 🗄️ Phase 1: MongoDB Atlas Database Provisioning (`edublitz`)

1. Connect to the **MongoDB Atlas Cluster** named `edublitz`:
   - Cluster Host: `edublitz.laegfsa.mongodb.net`
   - Application Name: `edublitz`
2. Create the Database User and Password:
   - **Username**: `linux`
   - **Password**: `redhat`
3. Verify the creation of the three microservice databases:
   - `catalog_db`
   - `cart_db`
   - `orders_db`
4. Add your EC2 VM server public/private IPs to the Atlas **Network Access / IP Access List**.
5. The MongoDB Atlas Connection URIs for each microservice are:

  **Catalog Service Connection URI**:
   ```env
   MONGODB_URI=mongodb+srv://naganeharshwardhan64_db_user:o7jGIfMnjqD7DcYm@edublitz.cjqyufm.mongodb.net/catalog_db?retryWrites=true&w=majority
   ```

   **Cart Service Connection URI**:
   ```env
  MONGODB_URI=naganeharshwardhan64_db_user:o7jGIfMnjqD7DcYm@edublitz.cjqyufm.mongodb.net/cart_db?retryWrites=true&w=majority
   ```

   **Order Service Connection URI**:
   ```env
  MONGODB_URI=mongodb+srv://naganeharshwardhan64_db_user:o7jGIfMnjqD7DcYm@edublitz.cjqyufm.mongodb.net/orders_db?retryWrites=true&w=majority
   ```

---

## ⚡ Phase 2: Deploying AWS Lambda Serverless Functions

### 1. Deploy `payment-service` (AWS Lambda 1)
```
aws iam create-role \
  --role-name ShopiTryLambdaRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Service": "lambda.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    ]
  }'

```
```
aws iam attach-role-policy \
  --role-name ShopiTryLambdaRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

```


```

```bash
# Navigate to payment service
cd backend/payment-service

# Install production dependencies only
npm install --omit=dev

# Create deployment package artifact
zip -r payment-service.zip src node_modules package.json

# Deploy to AWS Lambda via AWS CLI
aws lambda create-function \
  --function-name shopitry-payment-service \
  --runtime nodejs20.x \
  --role arn:aws:iam::501789774791:role/ShopiTryLambdaRole \
  --handler src/handler.handler \
  --zip-file fileb://payment-service.zip \
  --region ap-south-1

# Configure environment variables
aws lambda update-function-configuration \
  --function-name shopitry-payment-service \
  --environment "Variables={PAYMENT_GATEWAY_MODE=PRODUCTION}" \
  --region ap-south-1

---------------------------------------------------------------------------
aws lambda update-function-configuration \
  --function-name shopitry-payment-service \
  --environment "Variables={PAYMENT_GATEWAY_MODE=PRODUCTION,MONGODB_URI=mongodb+srv://naganeharshwardhan64_db_user:o7jGIfMnjqD7DcYm@edublitz.cjqyufm.mongodb.net/payment_db?retryWrites=true&w=majority}" \
  --region ap-south-1

--------------------------------------------------------------------------------
aws lambda invoke \
  --function-name shopitry-payment-service \
  --region ap-south-1 \
  response.json

cat response.json
---------------------------------------------------------------------------------
```

### 2. Deploy `notification-service` (AWS Lambda 2)
```
aws sns create-topic \
  --name aethercart-notifications \
  --region ap-south-1
```

```bash
# Navigate to notification service
cd backend/notification-service

# 1. Install production dependencies
npm install --omit=dev

# 2. Create the deployment package
zip -r notification-service.zip src node_modules package.json

# 3. Create the AWS Lambda function (using your correct region ap-south-1)
aws lambda create-function \
  --function-name shopitry-notification-service \
  --runtime nodejs20.x \
  --role arn:aws:iam::501789774791:role/ShopiTryLambdaRole \
  --handler src/handler.handler \
  --zip-file fileb://notification-service.zip \
  --region ap-south-1
----------------------------


aws sns subscribe \
  --topic-arn arn:aws:sns:ap-south-1:501789774791:aethercart-notifications \
  --protocol lambda \
  --notification-endpoint arn:aws:lambda:ap-south-1:501789774791:function:shopitry-notification-service \
  --region ap-south-1

```

---

## 🖥️ Phase 3: Deploying Core Microservices to AWS EC2 / VM Instances

### 1. Provision EC2 Instances
Launch 4 Ubuntu 22.04 LTS EC2 instances (or VM Servers):
- **VM Server 1**: `gateway-service` (Public subnet, Ports 80, 443, 5000)
- **VM Server 2**: `catalog-service` (Private subnet, Port 5001)
- **VM Server 3**: `cart-service` (Private subnet, Port 5002)
- **VM Server 4**: `order-service` (Private subnet, Port 5003)

### 2. Server Setup Commands (All VM Servers)
Execute on each VM server via SSH:
or give as user data.

```bash
#!/bin/bash

# Update system packages
sudo apt update -y
sudo apt install zip -y
git clone https://github.com/GitHarsh999/edublitz-e-commerce-application-shopitry.git
# Install Node.js 20 LTS & Nginx
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs git nginx

# Install PM2 process manager globally
sudo npm install -g pm2
```

### 3. Deploy Gateway Service (VM Server 1)

```bash
# Clone project repository
cd your-project/backend/gateway-service

# Install dependencies
npm install --omit=dev

# Configure production .env file
cat <<EOT > .env
PORT=5000
JWT_SECRET=shopitry_super_secret_jwt_key_2026
CATALOG_SERVICE_URL=http://3.111.40.44:5001
CART_SERVICE_URL=http://13.232.28.163:5002
ORDER_SERVICE_URL=http://15.207.14.255:5003
PAYMENT_SERVICE_URL=https:https://o7kgusixzf2i2xvjltcswfdua40ajzou.lambda-url.ap-south-1.on.aws/
NOTIFICATION_SERVICE_URL=https://zflaqgskqzblauthh2mu4kdnzm0cugck.lambda-url.ap-south-1.on.aws/
EOT

# Start gateway service using PM2
pm2 start src/index.js --name "gateway-service"
pm2 save
pm2 startup
```

### 4. Deploy Catalog Service (VM Server 2)

```bash
cd /var/www/shopitry/backend/catalog-service
npm install --omit=dev

cat <<EOT > .env
PORT=5001
 MONGODB_URI="mongodb+srv://naganeharshwardhan64_db_user:o7jGIfMnjqD7DcYm@edublitz.cjqyufm.mongodb.net/catalog_db?retryWrites=true&w=majority"
EOT

pm2 start src/index.js --name "catalog-service"
pm2 save
```

### 5. Deploy Cart Service (VM Server 3)

```bash
cd /var/www/shopitry/backend/cart-service
npm install --omit=dev

cat <<EOT > .env
PORT=5002
  MONGODB_URI=naganeharshwardhan64_db_user:o7jGIfMnjqD7DcYm@edublitz.cjqyufm.mongodb.net/cart_db?retryWrites=true&w=majority

EOT

pm2 start src/index.js --name "cart-service"
pm2 save
```

### 6. Deploy Order Service (VM Server 4)

```bash
cd /var/www/shopitry/backend/order-service
npm install --omit=dev

cat <<EOT > .env
PORT=5003



CATALOG_SERVICE_URL=http://3.111.40.44:5001
CART_SERVICE_URL=http://13.232.28.163:5002
ORDER_SERVICE_URL=http://15.207.14.255:5003
PAYMENT_SERVICE_URL=https:https://o7kgusixzf2i2xvjltcswfdua40ajzou.lambda-url.ap-south-1.on.aws/
NOTIFICATION_SERVICE_URL=https://zflaqgskqzblauthh2mu4kdnzm0cugck.lambda-url.ap-south-1.on.aws/


EOT

pm2 start src/index.js --name "order-service"
pm2 save
```

### 7. Configure Nginx Reverse Proxy & SSL on Gateway Server
On **VM Server 1** (`gateway-service`):

```bash

check it once from anoop repo.
# Create Nginx configuration
sudo vim /etc/nginx/sites-available/shopitry

# Paste configuration:
server {
    server_name api.shopitry.com;

    location / {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
# Enable site & restart Nginx
sudo ln -s /etc/nginx/sites-available/shopitry /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx

# SSL Certificate setup with Certbot
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d api.shopitry.com
```

---

## 🌐 Phase 4: Deploying React SPAs to AWS S3 & CloudFront

### 1. Deploy Customer Storefront (`frontend/storefront`)

```bash
# Build React storefront for production
cd frontend/storefront
npm install
npm run build

# Create S3 Bucket
aws s3 mb s3://shopitry-storefront-prod --region ap-south-1

# Enable static website hosting
aws s3 website s3://shopitry-storefront-prod/ --index-document index.html --error-document index.html

# Sync build files to S3 bucket
aws s3 sync dist/ s3://shopitry-storefront-prod --delete --acl public-read





------------------------------------------------------------------------
or
# Create the bucket (choose a globally unique name if shopitry-storefront-prod is already taken)
aws s3 mb s3://shopitry-storefront-prod1 --region ap-south-1

# Enable static website hosting
aws s3 website s3://shopitry-storefront-prod1/ --index-document index.html --error-document index.html

# Sync your build files (leaving out --acl public-read)
aws s3 sync dist/ s3://shopitry-storefront-prod1 --delete

-------------------------------------------------------------------------
```

### 2. Deploy Operations Admin Dashboard (`frontend/admin-dashboard`)

```bash
# Build React admin application for production
cd frontend/admin-dashboard
npm install
npm run build

# Create S3 Bucket
aws s3 mb s3://shopitry-admin-prod --region ap-south-1

# Enable static website hosting
aws s3 website s3://shopitry-admin-prod/ --index-document index.html --error-document index.html

# Sync build files to S3 bucket
aws s3 sync dist/ s3://shopitry-admin-prod --delete --acl public-read
-------------------------------------------------------------------------------------------
or
# 1. Install dependencies for the admin dashboard
npm install

# 2. Build the admin dashboard
npm run build

# 3. Create your S3 bucket (if you haven't already)
aws s3 mb s3://shopitry-admin-prod1 --region ap-south-1

# 4. Enable static website hosting
aws s3 website s3://shopitry-admin-prod1/ --index-document index.html --error-document index.html

# 5. Sync your build files to S3
aws s3 sync dist/ s3://shopitry-admin-prod1 --delete


```

---

## 🔍 Phase 5: Verification & Health Checks

Run these commands to verify that all deployment targets are online and healthy:

```bash
# 1. Test Gateway Health Endpoint
curl -i https://api.shopitry.com/health

# 2. Test Catalog Products API via Gateway
curl -i https://api.shopitry.com/api/catalog/products

# 3. Test Admin Authentication via Gateway
curl -i -X POST https://api.shopitry.com/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@shopitry.com","password":"adminpassword123"}'

# 4. Check PM2 status on VM Servers
pm2 status
pm2 logs --lines 50
```
