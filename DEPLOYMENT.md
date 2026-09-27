# ShopiTry AWS Deployment Guide

This guide deploys ShopiTry with one public gateway EC2, three private service EC2 instances, one command-center EC2 bastion, two Lambda Function URLs, MongoDB Atlas, and two S3 website buckets. It intentionally does not use CloudFront, a CDN, a custom domain, or SSL.

Because this setup uses S3 website endpoints and an HTTP gateway, it is suitable for testing only. Add HTTPS before handling real customer credentials or payment data.

## 1. Architecture

```text
Storefront S3 website ----\
                           > Public gateway EC2 :5000
Admin S3 website ---------/          |
                                     | private VPC traffic
                 +-------------------+-------------------+
                 |                   |                   |
           Catalog EC2 :5001   Cart EC2 :5002     Order EC2 :5003
                                                           |
                                               Payment and notification
                                                   Lambda Function URLs

Command-center EC2 -> SSH only -> gateway and private instances
All services -> MongoDB Atlas through the configured outbound route
```

The command-center instance is a bastion host. It is not part of the browser request path.

## 2. Prerequisites

Use one AWS region for EC2 and Lambda. Assign an Elastic IP to the gateway so its address remains stable.

```bash
export AWS_REGION=ap-south-1
export GATEWAY_IP=13.127.220.250
export STOREFRONT_BUCKET=shopitry-storefront-prod1
export ADMIN_BUCKET=shopitry-admin-prod1
```

## 3. VPC and Security Groups

Create one VPC with a public subnet for the gateway and private subnets for catalog, cart, and order. The private subnets need the required outbound route, normally through a NAT Gateway, to reach MongoDB Atlas and install packages.

Gateway security group:

```text
Inbound TCP 5000: 0.0.0.0/0
Inbound TCP 22: command-center security group only
Outbound: VPC and required internet/NAT traffic
```

Private-services security group:

```text
Inbound TCP 5001: gateway security group only
Inbound TCP 5002: gateway security group only
Inbound TCP 5003: gateway security group only
Inbound TCP 22: command-center security group only
Outbound: MongoDB Atlas, Lambda URLs, and required package traffic
```

Do not expose ports 5001, 5002, or 5003 to the internet. Use private IPs or private DNS names in service configuration. Use security-group references instead of hard-coded IP rules where possible.

## 4. MongoDB Atlas

Create these databases:

```text
catalog_db
cart_db
orders_db
```

Create a least-privilege database user. Store its URI only in the relevant EC2 `.env` file; never commit it to Git or this document.

```env
MONGODB_URI=mongodb+srv://naganeharshwardhan64_db_user:<mongodb-password>@edublitz.cjqyufm.mongodb.net/catalog_db?retryWrites=true&w=majority
```

If private instances access Atlas through a NAT Gateway, add the NAT Gateway Elastic IP to the Atlas IP access list. Atlas cannot whitelist a private `10.x.x.x` address after it has been translated through NAT.

## 5. Common EC2 Setup

Run on the gateway and all three private service instances:

```bash
sudo apt update
sudo apt install -y git curl
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm install --global pm2
sudo mkdir -p /var/www/shopitry
sudo chown -R "$USER":"$USER" /var/www/shopitry
git clone  https://github.com/GitHarsh999/edublitz-e-commerce-application-shopitry/ /var/www/shopitry
```

Replace `<repository-url>` with the repository URL. Do not place `.env` files in the repository.

## 6. Deploy Private Services

### Catalog EC2

```bash
cd /var/www/shopitry/backend/catalog-service
npm install --omit=dev
cat > .env <<'EOF'
PORT=5001
MONGODB_URI=mongodb+srv://naganeharshwardhan64_db_user:o7jGIfMnjqD7DcYm@edublitz.cjqyufm.mongodb.net/catalog_db?retryWrites=true&w=majority
EOF
pm2 start src/index.js --name catalog-service
pm2 save
```

### Cart EC2

```bash
cd /var/www/shopitry/backend/cart-service
npm install --omit=dev
cat > .env <<'EOF'
PORT=5002
MONGODB_URI=mongodb+srv://naganeharshwardhan64_db_user:o7jGIfMnjqD7DcYm@edublitz.cjqyufm.mongodb.net/cart_db?retryWrites=true&w=majority
EOF
pm2 start src/index.js --name cart-service
pm2 save
```

### Order EC2

The order service calls cart over the VPC and calls both Lambdas using complete HTTPS Function URLs. A Lambda URL is not an EC2 port.

```bash
cd /var/www/shopitry/backend/order-service
npm install --omit=dev
cat > .env <<'EOF'
PORT=5003
MONGODB_URI=mongodb+srv://naganeharshwardhan64_db_user:<mongodb-password>@edublitz.cjqyufm.mongodb.net/orders_db?retryWrites=true&w=majority
CART_SERVICE_URL=http://172.31.4.86:5002
PAYMENT_SERVICE_URL=https://hptzrap27cpwdjxflrkdfqtphm0uqspo.lambda-url.ap-south-1.on.aws
NOTIFICATION_SERVICE_URL=https://d34ysjxlvlocbdha54zhr73whe0qdcof.lambda-url.ap-south-1.on.aws
EOF
pm2 start src/index.js --name order-service
pm2 save
```

On each instance, enable PM2 on reboot using the command printed by:

```bash
pm2 startup
```

## 7. Deploy the Gateway EC2

The gateway is the only backend URL called by either browser application.

```bash
cd /var/www/shopitry/backend/gateway-service
npm install --omit=dev
cat > .env <<'EOF'
PORT=5000
JWT_SECRET=<long-random-secret>
CATALOG_SERVICE_URL=http://172.31.15.190:5001
CART_SERVICE_URL=http://172.31.4.86:5002
ORDER_SERVICE_URL=http://172.31.4.36:5003
PAYMENT_SERVICE_URL=https://hptzrap27cpwdjxflrkdfqtphm0uqspo.lambda-url.ap-south-1.on.aws
NOTIFICATION_SERVICE_URL=https://d34ysjxlvlocbdha54zhr73whe0qdcof.lambda-url.ap-south-1.on.aws
EOF
pm2 start src/index.js --name gateway-service
pm2 save
pm2 startup
```

The current gateway CORS middleware allows cross-origin requests. The gateway must be publicly reachable on TCP 5000. If you later restrict CORS, allow both S3 website origins.

## 8. Deploy Lambda Functions

### 8.1 Create the Lambda execution role

Run these commands once from a workstation with AWS CLI credentials that can manage IAM, Lambda, and SNS:

```bash
export AWS_REGION=ap-south-1
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

cat > lambda-trust-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "lambda.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name ShopiTryLambdaRole \
  --assume-role-policy-document file://lambda-trust-policy.json

aws iam attach-role-policy \
  --role-name ShopiTryLambdaRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

export LAMBDA_ROLE_ARN=arn:aws:iam::$AWS_ACCOUNT_ID:role/ShopiTryLambdaRole
```

If the role already exists, skip `create-role` and keep the existing role ARN. Wait briefly after creating the role before creating Lambda functions so IAM has time to propagate.

### 8.2 Package and create the Lambda functions

Run the following separately for each service:

```bash
cd backend/payment-service
npm install --omit=dev
zip -r payment-service.zip src node_modules package.json
aws lambda create-function \
  --function-name shopitry-payment-service \
  --runtime nodejs20.x \
  --role "$LAMBDA_ROLE_ARN" \
  --handler src/handler.handler \
  --zip-file fileb://payment-service.zip \
  --region "$AWS_REGION"

cd ../notification-service
npm install --omit=dev
zip -r notification-service.zip src node_modules package.json
aws lambda create-function \
  --function-name shopitry-notification-service \
  --runtime nodejs20.x \
  --role "$LAMBDA_ROLE_ARN" \
  --handler src/handler.handler \
  --zip-file fileb://notification-service.zip \
  --region "$AWS_REGION"
```

If either function already exists, use `aws lambda update-function-code` instead of `create-function`.

### 8.3 Create Lambda Function URLs

The gateway and order service call these HTTPS URLs. They are not EC2 ports 5004 or 5005.

```bash
aws lambda create-function-url-config \
  --function-name shopitry-payment-service \
  --auth-type NONE \
  --cors '{"AllowOrigins":["*"],"AllowMethods":["POST"],"AllowHeaders":["content-type"]}' \
  --region "$AWS_REGION"

aws lambda add-permission \
  --function-name shopitry-payment-service \
  --statement-id FunctionURLAllowPublicAccess \
  --action lambda:InvokeFunctionUrl \
  --principal '*' \
  --function-url-auth-type NONE \
  --region "$AWS_REGION"

aws lambda add-permission \
  --function-name shopitry-payment-service \
  --statement-id FunctionURLInvokeAllowPublicAccess \
  --action lambda:InvokeFunction \
  --principal '*' \
  --invoked-via-function-url \
  --region "$AWS_REGION"

aws lambda create-function-url-config \
  --function-name shopitry-notification-service \
  --auth-type NONE \
  --cors '{"AllowOrigins":["*"],"AllowMethods":["POST"],"AllowHeaders":["content-type"]}' \
  --region "$AWS_REGION"

aws lambda add-permission \
  --function-name shopitry-notification-service \
  --statement-id FunctionURLAllowPublicAccess \
  --action lambda:InvokeFunctionUrl \
  --principal '*' \
  --function-url-auth-type NONE \
  --region "$AWS_REGION"

aws lambda add-permission \
  --function-name shopitry-notification-service \
  --statement-id FunctionURLInvokeAllowPublicAccess \
  --action lambda:InvokeFunction \
  --principal '*' \
  --invoked-via-function-url \
  --region "$AWS_REGION"

aws lambda get-function-url-config \
  --function-name shopitry-payment-service --region "$AWS_REGION"
aws lambda get-function-url-config \
  --function-name shopitry-notification-service --region "$AWS_REGION"
```

Use the returned URLs in the gateway and order-service `.env` files as shown above. If a Function URL already exists, use `update-function-url-config` instead of `create-function-url-config`.

For Lambda Function URLs created through the AWS CLI, both invocation permissions are required. If the URL was already created, run these missing-permission commands before testing:

### 8.4 Create the SNS topic and connect notification Lambda

The notification handler can be called through its Function URL, and this SNS subscription also allows SNS to invoke it for notification events:

```bash
export SNS_TOPIC_ARN=$(aws sns create-topic \
  --name aethercart-notifications \
  --region "$AWS_REGION" \
  --query TopicArn --output text)

aws sns subscribe \
  --topic-arn "$SNS_TOPIC_ARN" \
  --protocol lambda \
  --notification-endpoint "arn:aws:lambda:$AWS_REGION:$AWS_ACCOUNT_ID:function:shopitry-notification-service" \
  --region "$AWS_REGION"

aws lambda add-permission \
  --function-name shopitry-notification-service \
  --statement-id AllowSnsInvoke \
  --action lambda:InvokeFunction \
  --principal sns.amazonaws.com \
  --source-arn "$SNS_TOPIC_ARN" \
  --region "$AWS_REGION"
```

Confirm the subscription:

```bash
aws sns list-subscriptions-by-topic \
  --topic-arn "$SNS_TOPIC_ARN" \
  --region "$AWS_REGION"
```

Test them directly before testing checkout:

```bash
curl -i -X POST "https://hptzrap27cpwdjxflrkdfqtphm0uqspo.lambda-url.ap-south-1.on.aws/api/payments/process" \
  -H "Content-Type: application/json" \
  -d '{"orderId":"test-order","amount":10,"currency":"USD","paymentMethod":"Card"}'

curl -i -X POST "https://d34ysjxlvlocbdha54zhr73whe0qdcof.lambda-url.ap-south-1.on.aws/api/notifications/send" \
  -H "Content-Type: application/json" \
  -d '{"type":"ORDER_CONFIRMATION","recipientEmail":"test@example.com","orderId":"test-order","totalAmount":10}'
```

## 9. Build and Host Frontends on S3

The React applications read `VITE_API_GATEWAY_URL` during the Vite build. This fixes the original problem where deployed pages called the visitor's `localhost:5000`.

### Storefront

Run from your workstation:

```bash
cd frontend/storefront
echo "VITE_API_GATEWAY_URL=http://13.127.220.250:5000" > .env.production
npm install
npm run build
aws s3 mb s3://shopitry-storefront-prod1 --region ap-south-1
aws s3 website s3://shopitry-storefront-prod1 --index-document index.html --error-document index.html
aws s3 sync dist/ s3://shopitry-storefront-prod1 --delete
```

### Admin dashboard

```bash

cd frontend/admin-dashboard
echo "VITE_API_GATEWAY_URL=http://13.127.220.250:5000" > .env.production
npm install
npm run build
aws s3 mb s3://shopitry-admin-prod1 --region ap-south-1
aws s3 website s3://shopitry-admin-prod1 --index-document index.html --error-document index.html
aws s3 sync dist/ s3://shopitry-admin-prod1 --delete
```



```
if products are not showing then add them via admin dashbord as  
Use these product values in Admin Dashboard → Product Catalog → Create Product SKU.

Name	Price	Category	Stock	Image URL
NovaSound Wireless Earbuds	79.99	Audio	80	https://images.unsplash.com/photo-1606220945770-b5b6c2c55bf1?auto=format&fit=crop&w=800&q=80

- you can used your own product image url from internet and add it.

```

------------------------------------------------------------------------------------------------------------------------

If you dont know how to make bucket public manually then use following way->

Configure each bucket for public website reads with a bucket policy allowing `s3:GetObject` on `arn:aws:s3:::<bucket>/*`. If Object Ownership is `BucketOwnerEnforced`, do not use `--acl public-read`.

For this direct S3 website setup, run the following after creating both buckets. This intentionally makes the website objects public; do not use this pattern for private data buckets.

```bash
for BUCKET in shopitry-storefront-prod1 shopitry-admin-prod1; do
  aws s3api put-public-access-block \
    --bucket "$BUCKET" \
    --public-access-block-configuration \
    BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false \
    --region ap-south-1

  cat > /tmp/${BUCKET}-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicReadForWebsite",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::$BUCKET/*"
  }]
}
EOF

  aws s3api put-bucket-policy \
    --bucket "$BUCKET" \
    --policy file:///tmp/${BUCKET}-policy.json \
    --region ap-south-1
done
```

Open the S3 website endpoint, not the REST bucket endpoint:

```text
http://<bucket>.s3-website-<aws-region>.amazonaws.com
```

The page and API are both HTTP in this no-SSL setup. Opening an HTTPS page while calling an HTTP API causes browser mixed-content blocking.

## 10. Verification

Run checks in this order.

From each service instance:

```bash
pm2 status
curl -i http://localhost:5001/health
curl -i http://localhost:5002/health
curl -i http://localhost:5003/health
```

Only the matching local port is expected to work on each instance.

From the gateway instance:

```bash
curl -i http://172.31.15.190:5001/health
curl -i http://172.31.4.86:5002/health
curl -i http://172.31.4.36:5003/health
curl -i http://localhost:5000/health
curl -i http://localhost:5000/api/catalog/products
```

From your workstation:

```bash
curl -i http://13.127.220.250:5000/health
curl -i http://13.127.220.250:5000/api/catalog/products
curl -i -X POST http://13.127.220.250:5000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@shopitry.com","password":"adminpassword123"}'
```

The products request must return HTTP 200 and JSON containing `data`.

If the gateway returns HTTP 503, inspect:

```bash
pm2 logs gateway-service --lines 100
```

A gateway 503 means it could not connect to a downstream service. Check the private IP, PM2 process, security group, network ACL, subnet route, and Lambda URL in that order.

## 11. Browser Troubleshooting

In browser developer tools, the request URL must be:

```text
http://13.127.220.250:5000/api/catalog/products
```

Interpret failures as follows:

- `localhost:5000`: an old frontend build is still in S3, or `.env.production` was missing during `npm run build`.
- CORS error: the gateway is unreachable or CORS was restricted.
- Mixed-content error: the page is HTTPS while the API is HTTP; use the S3 website endpoint.
- HTTP 503 JSON from the gateway: the browser reached the gateway, but the gateway could not reach a private service.
- HTTP 200 with empty data: catalog is running; investigate MongoDB or seed data separately.

After changing `VITE_API_GATEWAY_URL`, rebuild and sync both `dist/` directories. Vite does not read this variable at runtime.

## 12. Security Cleanup

- Rotate MongoDB credentials previously committed in `Dep.md` or shared in shell history.
- Use a strong random `JWT_SECRET` in the gateway `.env`.
- Restrict SSH to the command-center security group or administrator IP.
- Keep private service ports closed to the internet.
- Add HTTPS and a CDN before real customer use.

## 13. Deployment Mistakes and Permanent Fixes

These were the issues found during the first deployment. They are recorded here so the same failure is not misdiagnosed next time.

### Frontend used the wrong gateway URL

The frontend initially used `http://localhost:5000`. In a browser, `localhost` means the customer's computer, not the gateway EC2. One EC2 build also contained the malformed URL `http:/13.127.220.250:5000` with only one slash.

**Fix:** set the Vite variable before every production build and verify the generated bundle:

```bash
echo "VITE_API_GATEWAY_URL=http://13.127.220.250:5000" > .env.production
npm run build
grep -R -F "http://13.127.220.250:5000" dist/assets
grep -R -F "localhost:5000" dist/assets || true
aws s3 sync dist/ s3://<correct-bucket> --delete
```

The browser must call the public gateway IP. Private `172.31.x.x` addresses belong only in backend `.env` files.

### Backend health was tested from the wrong machine

`curl.exe` is a Windows command and is unavailable on Ubuntu. Testing `localhost` on the gateway proves only that the service is running locally; it does not prove browser access.

**Fix:** use `curl` on Ubuntu and test the public IP from an external Windows machine:

```bash
curl -i http://localhost:5000/health
```

```text
curl.exe -i http://13.127.220.250:5000/health
```

Both must return HTTP 200.

### MongoDB credentials were invalid

The catalog service reported `bad auth: authentication failed` and fell back to in-memory data.

**Fix:** reset or create the Atlas database user, use a correct URL-encoded password, test the URI directly, then restart with `pm2 restart catalog-service --update-env`.

### MongoDB was connected but the catalog database was empty

After authentication was fixed, the service correctly switched from seed memory to MongoDB. Because `catalog_db` contained no product documents, the API returned HTTP 200 with an empty `data` array.

**Fix:** seed the database once during initial provisioning, or deploy the catalog startup-seeding code. After the initial bootstrap, products should be created through the Admin Dashboard and stored in MongoDB.

### EC2 code was older than the workspace code

The workspace contained frontend and catalog fixes, but the EC2 copy did not contain all of them. For example, `grep -n "Seeded" src/index.js` returned no result on EC2.

**Fix:** deploy the changed source files to EC2 before restarting PM2, then verify the deployed file with `grep` or `git diff`. A local workspace edit does not change an already-cloned EC2 copy.

### PM2 processes were started more than once

`pm2 start` returned `Script already launched` because the process already existed.

**Fix:** use the existing process name and refresh its environment:

```bash
pm2 restart catalog-service --update-env
pm2 restart gateway-service --update-env
pm2 save
```

Run the PM2 startup command once per instance so services return after reboot.

### Markdown code fences were pasted into Bash

The Bash `>` prompt appeared after the closing Markdown backticks were pasted into the terminal. Bash was waiting for unfinished input; it was not an application error.

**Fix:** paste only the commands inside the code block. Press `Ctrl+C` to cancel an accidental unfinished paste.

### Lambda Function URLs returned HTTP 403

The Function URLs were configured with `AuthType=NONE`, but AWS CLI-created URLs still required both `lambda:InvokeFunctionUrl` and `lambda:InvokeFunction` resource permissions.

**Fix:** add both permission statements. The final curl tests returned HTTP 200 for payment and notification Lambdas.

### S3 website hosting was incomplete

Creating a bucket and syncing files does not automatically make a direct S3 website public.

**Fix:** configure website hosting, disable the relevant Block Public Access settings, add a public `s3:GetObject` bucket policy, and open the S3 website endpoint rather than the REST bucket endpoint.

### Final verified state

The deployment is considered healthy when all of these are true:

```text
Gateway public health: HTTP 200
Catalog health: HTTP 200 and mongoConnected=true
Catalog products: HTTP 200 with count greater than zero
Cart health: HTTP 200
Order health: HTTP 200
Payment Lambda: HTTP 200
Notification Lambda: HTTP 200
Frontend bundle: contains http://13.127.220.250:5000 and no localhost:5000
```
