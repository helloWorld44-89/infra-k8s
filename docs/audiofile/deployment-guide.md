---
title: AudioFile Deployment Guide
date: 2026-05-29
author: Hermes Docs Agent
tags: [deployment, docker, infrastructure, kubernetes]
---

# AudioFile Deployment Guide

## Overview

AudioFile can be deployed using Docker containers for easy setup and scalability. This guide covers various deployment methods including Docker Compose, Kubernetes, and cloud platforms.

## Prerequisites

### System Requirements

- **CPU**: 2+ cores recommended
- **Memory**: 4GB+ RAM recommended
- **Storage**: 10GB+ for audiobook library
- **Network**: Port 8080 available
- **OS**: Linux, macOS, or Windows (WSL2)

### Software Dependencies

- **Docker**: Docker Engine 20.10+
- **Docker Compose**: 2.0+ (optional)
- **Git**: For cloning repository
- **curl**: For health checks

## Deployment Methods

### Method 1: Docker Compose (Recommended)

Quick and easy deployment for most users.

#### 1. Clone Repository

```bash
git clone https://github.com/helloWorld44-89/AudioFile.git
cd AudioFile
```

#### 2. Configure Environment

Create a `.env` file:

```bash
cat > .env << EOF
DATABASE_URL=sqlite:///config/audiofile.db
REDIS_URL=redis://redis:6379/0
LIBRARY_PATH=/audiobooks
INBOX_PATH=/inbox
LOGS_PATH=/logs
TZ=UTC
PUID=1000
PGID=1000
EOF
```

#### 3. Create Directories

```bash
mkdir -p audiobooks inbox config logs
chmod -R 755 audiobooks inbox config logs
```

#### 4. Deploy with Docker Compose

```bash
cd docker
docker-compose up -d
```

#### 5. Check Status

```bash
# Check logs
docker-compose logs -f

# Check health
curl http://localhost:8080/health

# Check containers
docker-compose ps
```

#### 6. Access AudioFile

Open browser to: `http://localhost:8080`

### Method 2: Manual Docker Deployment

For custom configurations or cloud deployments.

#### 1. Build Image

```bash
# Clone repository
git clone https://github.com/helloWorld44-89/AudioFile.git
cd AudioFile

# Build image
docker build -t audiofile:latest .

# Build multi-arch image
docker buildx build --platform linux/amd64,linux/arm64 -t audiofile:latest .
```

#### 2. Run Container

```bash
docker run -d \
  --name audiofile \
  -p 8080:8080 \
  -v $(pwd)/audiobooks:/library \
  -v $(pwd)/config:/config \
  -v $(pwd)/inbox:/inbox \
  -v $(pwd)/logs:/logs \
  -e DATABASE_URL=sqlite:///config/audiofile.db \
  -e REDIS_URL=redis://redis:6379/0 \
  -e LIBRARY_PATH=/library \
  -e INBOX_PATH=/inbox \
  -e LOGS_PATH=/logs \
  -e PUID=1000 \
  -e PGID=1000 \
  --restart unless-stopped \
  audiofile:latest
```

#### 3. With External Redis

```bash
docker run -d \
  --name audiofile \
  -p 8080:8080 \
  -v $(pwd)/audiobooks:/library \
  -v $(pwd)/config:/config \
  -v $(pwd)/inbox:/inbox \
  -v $(pwd)/logs:/logs \
  -e DATABASE_URL=sqlite:///config/audiofile.db \
  -e REDIS_URL=redis://host.docker.internal:6379/0 \
  -e LIBRARY_PATH=/library \
  -e INBOX_PATH=/inbox \
  -e LOGS_PATH=/logs \
  --restart unless-stopped \
  audiofile:latest
```

### Method 3: Kubernetes Deployment

For production environments with orchestration.

#### 1. Create Namespace

```bash
kubectl create namespace audiofile
```

#### 2. Redis Deployment

```yaml
# redis-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: audiofile
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        volumeMounts:
        - name: redis-data
          mountPath: /data
      volumes:
      - name: redis-data
        persistentVolumeClaim:
          claimName: redis-pvc
```

#### 3. AudioFile Deployment

```yaml
# audiofile-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: audiofile
  namespace: audiofile
spec:
  replicas: 2
  selector:
    matchLabels:
      app: audiofile
  template:
    metadata:
      labels:
        app: audiofile
    spec:
      containers:
      - name: audiofile
        image: apexvoid/audiofile:latest
        ports:
        - containerPort: 8080
        env:
        - name: DATABASE_URL
          value: "sqlite:///config/audiofile.db"
        - name: REDIS_URL
          value: "redis://redis:6379/0"
        - name: LIBRARY_PATH
          value: "/library"
        - name: INBOX_PATH
          value: "/inbox"
        - name: LOGS_PATH
          value: "/logs"
        - name: TZ
          value: "UTC"
        volumeMounts:
        - name: library
          mountPath: /library
        - name: config
          mountPath: /config
        - name: inbox
          mountPath: /inbox
        - name: logs
          mountPath: /logs
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: library
        persistentVolumeClaim:
          claimName: library-pvc
      - name: config
        persistentVolumeClaim:
          claimName: config-pvc
      - name: inbox
        persistentVolumeClaim:
          claimName: inbox-pvc
      - name: logs
        persistentVolumeClaim:
          claimName: logs-pvc
```

#### 4. Persistent Volume Claims

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pvc
  namespace: audiofile
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: library-pvc
  namespace: audiofile
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: config-pvc
  namespace: audiofile
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: inbox-pvc
  namespace: audiofile
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: logs-pvc
  namespace: audiofile
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

#### 5. Service and Ingress

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: audiofile
  namespace: audiofile
spec:
  selector:
    app: audiofile
  ports:
  - port: 80
  targetPort: 8080
  type: LoadBalancer
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: audiofile
  namespace: audiofile
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - audiofile.yourdomain.com
    secretName: audiofile-tls
  rules:
  - host: audiofile.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: audiofile
            port:
              number: 80
```

#### 6. Deploy

```bash
kubectl apply -f redis-deployment.yaml
kubectl apply -f audiofile-deployment.yaml
kubectl apply -f pvc.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
```

### Method 4: Cloud Platform Deployments

#### AWS ECS

```bash
# Task Definition JSON
cat > audiofile-task.json << EOF
{
  "family": "audiofile",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "audiofile",
      "image": "apexvoid/audiofile:latest",
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "DATABASE_URL",
          "value": "sqlite:///config/audiofile.db"
        },
        {
          "name": "REDIS_URL",
          "value": "redis://redis-cluster:6379/0"
        },
        {
          "name": "LIBRARY_PATH",
          "value": "/library"
        }
      ],
      "mountPoints": [
        {
          "sourceVolume": "library",
          "containerPath": "/library"
        },
        {
          "sourceVolume": "config",
          "containerPath": "/config"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/audiofile",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ],
  "volumes": [
    {
      "name": "library",
      "host": {
        "sourcePath": "/ecs/audiofile/library"
      }
    },
    {
      "name": "config",
      "host": {
        "sourcePath": "/ecs/audiofile/config"
      }
    }
  ]
}
EOF

# Register task definition
aws ecs register-task-definition --cli-input-json file://audiofile-task.json

# Create service
aws ecs create-service \
  --cluster audiofile-cluster \
  --service-name audiofile \
  --task-definition audiofile \
  --desired-count 2 \
  --launch-type FARGATE
```

#### Google Cloud Run

```bash
# Build and deploy
gcloud builds submit --tag gcr.io/your-project/audiofile
gcloud run deploy audiofile \
  --image gcr.io/your-project/audiofile \
  --platform managed \
  --region us-central1 \
  --memory 1Gi \
  --port 8080 \
  --set-env-vars "DATABASE_URL=sqlite:///config/audiofile.db" \
  --set-env-vars "REDIS_URL=redis://redis:6379/0" \
  --set-env-vars "LIBRARY_PATH=/library" \
  --allow-unauthenticated
```

## Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `DATABASE_URL` | `sqlite:///config/audiofile.db` | Database connection string |
| `REDIS_URL` | `redis://redis:6379/0` | Redis connection string |
| `LIBRARY_PATH` | `/library` | Audiobook library directory |
| `INBOX_PATH` | `/inbox` | Import directory |
| `LOGS_PATH` | `/logs` | Log directory |
| `TZ` | `UTC` | System timezone |
| `PUID` | `1000` | User ID for file permissions |
| `PGID` | `1000` | Group ID for file permissions |
| `LOG_LEVEL` | `INFO` | Logging level (DEBUG, INFO, WARNING, ERROR) |
| `MAX_CONVERSIONS` | `1` | Maximum concurrent conversions |
| `AUTO_IMPORT` | `true` | Enable auto-import |

### Volume Mounts

| Path | Purpose | Permissions |
|------|---------|-------------|
| `/library` | Audiobook storage | Read/Write |
| `/config` | Database and config | Read/Write |
| `/inbox` | Import processing | Read/Write |
| `/logs` | Application logs | Read/Write |

### Nginx Configuration

For production deployments with SSL termination:

```nginx
# /etc/nginx/sites-available/audiofile
server {
    listen 443 ssl http2;
    server_name audiofile.yourdomain.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /api {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /health {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
    }
}
```

## Monitoring and Maintenance

### Health Checks

```bash
# Check health
curl http://localhost:8080/health

# Check database
curl http://localhost:8080/api/system/info

# Check logs
tail -f logs/audiofile.log
```

### Log Management

```bash
# View logs
docker logs audiofile

# Follow logs
docker logs -f audiofile

# Export logs
docker logs audiofile > audiofile-logs.txt
```

### Backup and Recovery

#### Database Backup

```bash
# Create backup
cp config/audiofile.db backup/audiofile-$(date +%Y%m%d).db

# Restore backup
cp backup/audiofile-20260529.db config/audiofile.db
```

#### Library Backup

```bash
# Backup entire library
tar -czf audiofile-library-$(date +%Y%m%d).tar.gz library/

# Restore library
tar -xzf audiofile-library-20260529.tar.gz
```

#### Automated Backup Script

```bash
#!/bin/bash
# backup.sh

DATE=$(date +%Y%m%d)
BACKUP_DIR="/backups/audiofile"

# Create backup directory
mkdir -p $BACKUP_DIR

# Backup database
cp config/audiofile.db $BACKUP_DIR/audiofile-db-$DATE.db

# Backup library
tar -czf $BACKUP_DIR/audiofile-library-$DATE.tar.gz library/

# Keep only last 7 days
find $BACKUP_DIR -name "*.db" -mtime +7 -delete
find $BACKUP_DIR -name "*.tar.gz" -mtime +7 -delete

echo "Backup completed: $DATE"
```

### Performance Tuning

#### Resource Limits

```bash
# Docker resource limits
docker run -d \
  --name audiofile \
  --cpus=2 \
  --memory=4g \
  --memory-swap=4g \
  --pids-limit=100 \
  audiofile:latest
```

#### Conversion Optimization

```bash
# Increase conversion parallelism
export MAX_CONVERSIONS=2

# Use higher quality for better performance
export CONVERSION_QUALITY=high
```

## Security Configuration

### SSL/TLS Setup

```bash
# Generate self-signed certificate
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes

# Configure HTTPS
export SSL_CERT=/path/to/cert.pem
export SSL_KEY=/path/to/key.pem
```

### Authentication Setup

```bash
# Enable OIDC authentication
export OIDC_ENABLED=true
export OIDC_CLIENT_ID=your-client-id
export OIDC_CLIENT_SECRET=your-client-secret
export OIDC_DISCOVERY_URL=https://your-auth-provider/.well-known/openid-configuration
export OIDC_REDIRECT_URI=http://localhost:8080/auth/callback
```

### Network Security

```bash
# Firewall rules
sudo ufw allow 8080/tcp
sudo ufw enable

# Reverse proxy with SSL
sudo apt install nginx
sudo systemctl enable nginx
```

## Troubleshooting

### Common Issues

#### Container Won't Start

```bash
# Check logs
docker logs audiofile

# Check resource usage
docker stats audiofile

# Restart container
docker restart audiofile
```

#### Database Issues

```bash
# Check database file
ls -la config/audiofile.db

# Check database permissions
ls -la config/

# Rebuild database
rm config/audiofile.db
docker restart audiofile
```

#### Import Problems

```bash
# Check inbox permissions
ls -la inbox/

# Check file formats
file inbox/*.aax

# Check disk space
df -h
```

### Performance Issues

```bash
# Monitor CPU usage
top -p $(pgrep -f audiofile)

# Monitor memory usage
free -h

# Monitor disk I/O
iostat -x 1 5
```

### Network Issues

```bash
# Check port availability
netstat -tlnp | grep 8080

# Test connectivity
curl -v http://localhost:8080/health

# Check DNS resolution
nslookup audiofile.yourdomain.com
```

## Scaling

### Horizontal Scaling

#### Docker Compose

```yaml
# docker-compose.scale.yml
version: '3.8'
services:
  audiofile:
    image: apexvoid/audiofile:latest
    deploy:
      replicas: 3
    environment:
      - REDIS_URL=redis://redis:6379/0
    volumes:
      - ./library:/library
      - ./config:/config
```

#### Kubernetes

```yaml
# scale-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: audiofile
spec:
  replicas: 5
  # ... rest of config
```

### Load Balancing

#### Nginx Load Balancer

```nginx
upstream audiofile_backend {
    server audiofile1:8080;
    server audiofile2:8080;
    server audiofile3:8080;
}

server {
    listen 80;
    location / {
        proxy_pass http://audiofile_backend;
    }
}
```

### Storage Scaling

#### Distributed Storage

```bash
# NFS setup
sudo apt install nfs-kernel-server
sudo mkdir /nfs/audiofile
sudo chmod -R 755 /nfs/audiofile
sudo chown -R nobody:nogroup /nfs/audiofile

# /etc/exports
/nfs/audiofile 192.168.1.0/24(rw,sync,no_subtree_check)
```

## Migration

### From Previous Versions

1. **Backup existing data**
2. **Update Docker image**
3. **Migrate database schema**
4. **Test functionality**
5. **Update configuration**

### Data Migration

```bash
# Export database
sqlite3 config/audiofile.db ".backup backup/audiofile-v1.db"

# Import to new version
sqlite3 config/audiofile.db ".restore backup/audiofile-v1.db"
```

---

*This deployment guide is part of the AudioFile project. For the latest updates and additional configurations, visit the project repository.*