# React Cappuccino App CI/CD Pipeline - AWS Docker Deployment
![[react-app.png]]
This repository documents the full CI/CD pipeline implementation of my React-based Cappuccino application using GitHub Actions, Docker, and AWS EC2. The focus of this project is to demonstrate a hands-on, automated DevOps workflow.

## 📦 Project Overview

- **Frontend**: React.js
    
- **Containerization**: Docker
    
- **CI/CD Pipeline**: GitHub Actions
    
- **Deployment Target**: Amazon EC2 (Free Tier)
    
- **Web Server**: NGINX (reverse proxy)
    
- **Service Management**: systemd
    
- **Monitoring**: Amazon CloudWatch Agent
    

## 🗂️ Project Structure

```bash
.
├── .github
│   └── workflows
│       └── deploy.yml
├── nginx
│   └── default.conf
├── react-app
│   └── # Your React application code
├── Dockerfile
└── README.md
```

---

## 🚀 CI/CD Pipeline Workflow

### GitHub Actions Workflow: `.github/workflows/deploy.yml`

```yaml
name: Deploy React App to AWS EC2 via Docker

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push Docker image
        run: |
          docker build -t ${{ secrets.DOCKER_USERNAME }}/cappuccino-app:latest .
          docker push ${{ secrets.DOCKER_USERNAME }}/cappuccino-app:latest

      - name: Deploy to EC2
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ec2-user # (ubuntu in my case)  
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            sudo docker pull ${{ secrets.DOCKER_USERNAME }}/cappuccino-app:latest
            sudo docker stop cappuccino || true
            sudo docker rm cappuccino || true
            sudo docker run -d --name react-app -p 3000:80 ${{ secrets.DOCKER_USERNAME }}/react-app:latest
```

---

## 🐳 Docker Setup

### Dockerfile

```Dockerfile
# Stage 1 - Build

FROM node:20 as builder

WORKDIR /app

COPY . .

RUN npm install && npm run build

  

# Stage 2 - Serve

FROM nginx:alpine

COPY --from=builder /app/dist /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### `nginx/default.conf`

```nginx
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://localhost:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Enable nginx it:

```bash
sudo ln -s /etc/nginx/sites-available/react-app /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## 🔧 EC2 Configuration

### SSH Access

- Add your public SSH key to the EC2 instance.
    
- Ensure port 22 is temporarily open (`0.0.0.0/0`) for GitHub Actions testing.
### Install Docker

```bash
sudo apt update -y
sudo apt install -y docker
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user
```

## ⚙️ systemd Service File (Optional)

Create `/etc/systemd/system/react-app.service`:

```ini
[Unit]
Description=React App Docker Container
After=docker.service
Requires=docker.service

[Service]
Restart=always
ExecStart=/usr/bin/docker run --name react-app -p 3000:80 <your-dockerhub-username>/react-app:latest
ExecStop=/usr/bin/docker stop react-app
ExecStopPost=/usr/bin/docker rm react-app

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable react-app
sudo systemctl start react-app
sudo systemctl status react-app
```

---

## 📊 Install Amazon CloudWatch Agent

### Step 1: Download and install

```bash
sudo apt install amazon-cloudwatch-agent
```

### Step 2: Create CloudWatch config file (`/opt/aws/amazon-cloudwatch-agent/bin/config.json`)

Example:

```json
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "root"
  },
  "metrics": {
    "namespace": "EC2/DevOpsProject",
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}"
    },
    "aggregation_dimensions": [["InstanceId"]],
    "metrics_collected": {
      "cpu": {
        "measurement": ["cpu_usage_idle", "cpu_usage_iowait", "cpu_usage_user", "cpu_usage_system"],
        "metrics_collection_interval": 60
      },
      "mem": {
        "measurement": ["mem_used_percent"],
        "metrics_collection_interval": 60
      },
      "disk": {
        "measurement": ["used_percent"],
        "metrics_collection_interval": 60,
        "resources": ["*"]
      }
    }
  }
}

```

### Step 3: Start CloudWatch Agent

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json \
  -s
```
### Step 4: Enable the Agent on Boot
```bash
sudo systemctl enable amazon-cloudwatch-agent
```

---

## ✅ Final Checklist

---

## 🧠 Lessons Learned

- Managing secrets securely using GitHub Actions
    
- Handling dynamic IPs in security groups
    
- Automating SSH + Docker deployments
    
- Configuring `systemd` for persistent services
    
- Integrating CloudWatch for monitoring

---

## 📝 Author

**Jorge Bayuelo**  
DevOps Enthusiast | React Fan | Coffee Aficionado ☕

---

## 📎 License

This project is licensed under the MIT License.