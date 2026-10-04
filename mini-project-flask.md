# Day 7 — Flask → Docker → ECR → EC2

## Project Overview

Built and deployed a simple Flask application end-to-end using Docker and AWS.

### Architecture

```text
Developer Laptop
      │
      ├── Flask Application
      │
      ├── Dockerfile
      │      ↓
      ├── Docker Image
      │      ↓
      └── Docker Container
             │
             ↓ docker push
        AWS ECR Repository
             │
             ↓ docker pull
          AWS EC2
             │
       IAM Role + ECR ReadOnly
             │
             ↓
       Docker Container
       EC2 :80 → Container :5000
             │
             ↓
          Internet
             │
          Browser
```

## 1. Flask Application

Project structure:

```text
flask/
├── app.py
├── requirements.txt
└── Dockerfile
```

It was first tested locally with:

```bash
curl http://localhost:5000
```

## 2. Docker

Created a `Dockerfile` to package the application and its dependencies.

Key workflow:

```bash
docker build -t proj .
docker run -d --name proj -p 8080:5000 proj
```
Flask application continues listening on port `5000`, while users access the application through HTTP port `80`.


## 3. AWS ECR

Created an ECR repository and pushed the Docker image.

### ECR workflow

```text
Local Docker Image
       ↓
docker tag
       ↓
ECR Image URI
       ↓
docker push
       ↓
ECR Repository
```

Core commands:

```bash
aws ecr create-repository   --repository-name proj   --region ap-south-1

aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com

docker tag proj:latest <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/proj:latest

docker push <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/proj:latest
```

### Why `docker tag`?

The local image is:

```text
proj:latest
```

The tag gives it the ECR destination:

```text
<ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/proj:latest
```

`docker push` then uploads that image to ECR. It does not create any new image.

## 5. EC2 + IAM

An Ubuntu EC2 instance was launched and accessed using SSH.

Instead of storing AWS access keys on EC2, an IAM role was attached:

```text
EC2-ECR-ReadOnly-Role
        ↓
AmazonEC2ContainerRegistryReadOnly
```

This allows EC2 to pull images from ECR securely.

Verification:

```bash
aws sts get-caller-identity
```

Expected identity:

```text
assumed-role/EC2-ECR-ReadOnly-Role/...
```

### Secure authentication flow

```text
EC2
 ↓
IAM Role
 ↓
Temporary AWS credentials
 ↓
AWS CLI
 ↓
ECR
```

No permanent AWS access keys are required on EC2.

## 6. Pull and Run on EC2

After installing AWS CLI and Docker:

```bash
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com
```

Pull the image:

```bash
docker pull <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/proj:latest
```

Run it:

```bash
docker run -d   --name proj   -p 80:5000   <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/proj:latest
```

Verify:

```bash
docker ps
```

Expected:

```text
0.0.0.0:80->5000/tcp
```

The EC2 Security Group allowed:

```text
SSH   TCP 22  → My IP
HTTP  TCP 80  → 0.0.0.0/0
```

The application was finally verified externally:

```bash
curl http://<EC2-PUBLIC-IP>
```

Response:

```text
Application is running!
```

## Challenges Faced & What I Learnt


### 1. Container port vs host port

Initially tested the wrong port. The mapping:

```text
8080:5000
```

means:

```text
localhost:8080 → container:5000
```

Later used:

```text
80:5000
```

for EC2.

**Lesson:** Always read the `HOST:CONTAINER` order in `-p`.

### 2. EC2 SSH issues

Initially tried using a private IP and later encountered a key mismatch. A new Ubuntu instance with a public IPv4 address was launched.

**Lesson:** Internet SSH requires a reachable public IP and the correct key pair/user.

For Ubuntu:

```bash
ssh -i key.pem ubuntu@PUBLIC_IP
```

### 3. AWS CLI credentils on EC2

Old credentials in `~/.aws/credentials` caused:

```text
InvalidClientTokenId
```

After removing them, the EC2 IAM role was used correctly.

**Lesson:** When using an EC2 IAM role, don't configure permanent AWS access keys on the instance.

### 4. Docker permission issue

The `ubuntu` user initially could not access `/var/run/docker.sock`.

Fixed with:

```bash
sudo usermod -aG docker ubuntu
```

After reconnecting:

```bash
docker ps
```

worked without `sudo`.

### 5. ECR login vs Docker permissions

Docker was authenticated as the `ubuntu` user, but `sudo docker pull` used root's Docker configuration and therefore had no ECR credentials.

**Lesson:** Avoid mixing `docker` and `sudo docker`. Configure the user to access Docker and consistently use `docker`.

## Takeaways

Able to explain:

- Why GitHub and ECR serve different purposes.
- Why Docker images are tagged with the ECR repository URI.
- Difference between `docker push` and `docker pull`.
- Why EC2 needs an IAM role to securely pull from ECR.
- Why an IAM role is preferred over storing AWS access keys on EC2.
- How Security Groups control inbound traffic.
- What `-p 80:5000` means.
- Difference between an image and a running container.
- How traffic reaches the Flask application.

### Final Flow

```text
Code
 ↓
GitHub
 ↓
Dockerfile
 ↓
Docker Image
 ↓
ECR
 ↓
EC2 IAM Role
 ↓
EC2
 ↓
Docker Container
 ↓
Port 80 → Port 5000
 ↓
Flask Application
 ↓
Browser
```
