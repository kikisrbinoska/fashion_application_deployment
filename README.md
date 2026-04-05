# Fashion Application

## Overview

This document provides comprehensive documentation for the Fashion Application, a full-stack enterprise solution featuring a Spring Boot backend, modern web frontend, and PostgreSQL database with primary-replica replication architecture.

## Table of Contents

- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Local Development](#local-development)
- [Kubernetes Deployment](#kubernetes-deployment)
- [Configuration](#configuration)
- [Database Architecture](#database-architecture)
- [Monitoring and Health](#monitoring-and-health)
- [Troubleshooting](#troubleshooting)

## Architecture

### Components

- **Frontend**: React/Angular application served by Nginx
- **Backend**: Spring Boot application (Java)
- **Database**: PostgreSQL 15 with streaming replication (1 primary + 2 replicas)
- **Proxy**: Nginx reverse proxy for API routing

### Network Architecture

```
Internet → Ingress (fashion.local) → Frontend Service (NodePort 30080)
                                   → Backend Service (ClusterIP 8080)
                                   → PostgreSQL Services:
                                       - postgres-primary (writes)
                                       - postgres-readonly (reads)
```

## Prerequisites

### Local Development Requirements
- Docker Engine 20.10+
- Docker Compose 2.0+
- 4GB RAM minimum

### Kubernetes Deployment Requirements
- Kubernetes cluster 1.24+
- kubectl configured
- Nginx Ingress Controller installed
- 8GB RAM minimum for cluster
- Storage provisioner for PersistentVolumes

## Local Development Environment

### Quick Start Guide

1. **Clone the repository**
   ```bash
   git clone <repository-url>
   cd <repository-directory>
   ```

2. **Start all services**
   ```bash
   docker-compose up -d
   ```

3. **Access the application**
   - Frontend: http://localhost
   - Backend API: http://localhost:8080
   - Database: localhost:5432

### Services

| Service | Port | Container Name | Description |
|---------|------|----------------|-------------|
| Frontend | 80 | fashion-frontend | Web application |
| Backend | 8080 | fashion-backend | REST API |
| Database | 5432 | fashion-db | PostgreSQL |

### Docker Compose Commands

```bash
# Start services
docker-compose up -d

# View logs
docker-compose logs -f [service-name]

# Stop services
docker-compose down

# Rebuild and restart
docker-compose up -d --build

# Remove volumes (clean database)
docker-compose down -v
```

### Database Access

```bash
# Connect to PostgreSQL
docker exec -it fashion-db psql -U postgres -d postgres

# View database credentials
# Username: postgres
# Password: kikis
# Database: postgres
```

## Kubernetes Deployment

### Installation Procedure

1. **Create namespace and secrets**
   ```bash
   kubectl apply -f namespace.yaml
   kubectl apply -f secret.yaml
   ```

2. **Deploy database layer**
   ```bash
   kubectl apply -f configmap.yaml  # db-config
   kubectl apply -f service.yaml    # postgres services
   kubectl apply -f statefulset.yaml
   ```

3. **Wait for database initialization**
   ```bash
   kubectl wait --for=condition=ready pod/postgres-0 -n fashion-app --timeout=300s
   kubectl wait --for=condition=ready pod/postgres-1 -n fashion-app --timeout=300s
   kubectl wait --for=condition=ready pod/postgres-2 -n fashion-app --timeout=300s
   ```

4. **Deploy backend**
   ```bash
   kubectl apply -f configmap.yaml  # backend-config
   kubectl apply -f deployment.yaml # backend deployment
   kubectl apply -f service.yaml    # backend-service
   ```

5. **Deploy frontend**
   ```bash
   kubectl apply -f nginx-configmap.yaml
   kubectl apply -f deployment.yaml  # frontend deployment
   kubectl apply -f service.yaml     # frontend-service
   ```

6. **Configure ingress**
   ```bash
   kubectl apply -f ingress.yaml
   ```

7. **Add host entry**
   ```bash
   # Add to /etc/hosts (Linux/Mac) or C:\Windows\System32\drivers\etc\hosts (Windows)
   <ingress-ip> fashion.local
   ```

### Verification

```bash
# Check all pods
kubectl get pods -n fashion-app

# Check services
kubectl get svc -n fashion-app

# Check ingress (verify it's working and has an IP address)
kubectl get ingress -n fashion-app
# Expected output should show ADDRESS column with an IP
# NAME              CLASS   HOSTS           ADDRESS        PORTS   AGE
# fashion-ingress   nginx   fashion.local   <EXTERNAL-IP>  80      1m

# View ingress details
kubectl describe ingress fashion-ingress -n fashion-app

# View logs
kubectl logs -f deployment/fashion-backend -n fashion-app
kubectl logs -f deployment/fashion-frontend -n fashion-app
kubectl logs -f postgres-0 -n fashion-app
```

## Configuration


### Scaling

#### Backend Scaling
```bash
# Current: 2 replicas
kubectl scale deployment fashion-backend --replicas=3 -n fashion-app
```

#### Frontend Scaling
```bash
# Current: 2 replicas
kubectl scale deployment fashion-frontend --replicas=3 -n fashion-app
```

#### Database Replicas
```bash
# Current: 3 replicas (1 primary + 2 read replicas)
# To add more read replicas:
kubectl scale statefulset postgres --replicas=4 -n fashion-app
```

### Resource Limits

| Component | Request CPU | Request Memory | Limit CPU | Limit Memory |
|-----------|-------------|----------------|-----------|--------------|
| Backend | 250m | 512Mi | 500m | 1Gi |
| Frontend | 100m | 128Mi | 200m | 256Mi |
| Database | 250m | 512Mi | 500m | 1Gi |

## Database Architecture

### Primary-Replica Setup

- **postgres-0**: Primary node (read/write)
- **postgres-1**: Replica node (read-only)
- **postgres-2**: Replica node (read-only)

### Replication Configuration

**PostgreSQL Streaming Replication**
- WAL level: replica
- Max WAL senders: 10
- Max replication slots: 10
- Hot standby enabled on replicas

### Connection Routing

- **Write operations**: `postgres-primary` service → postgres-0
- **Read operations**: `postgres-readonly` service → load balanced across all replicas
- **Replication**: Direct pod-to-pod via headless service

### Replication User
```
Username: replicator
Password: replica2026 (from db-secret)
```

### Checking Replication Status

```bash
# Connect to primary
kubectl exec -it postgres-0 -n fashion-app -- psql -U postgres

# Check replication status
SELECT * FROM pg_stat_replication;

# Check if server is primary or replica
SELECT pg_is_in_recovery();
# false = primary, true = replica
```

## Monitoring and Health

### Health Checks

#### Database
```bash
# Liveness probe: pg_isready every 10s
# Readiness probe: pg_isready + SELECT 1 query

# Manual health check
kubectl exec -it postgres-0 -n fashion-app -- pg_isready -U postgres
```

#### Backend
```bash
# Check backend health
curl http://backend-service.fashion-app.svc.cluster.local:8080/actuator/health
```

### Logs

```bash
# Backend logs
kubectl logs -f deployment/fashion-backend -n fashion-app

# Frontend logs
kubectl logs -f deployment/fashion-frontend -n fashion-app

# Database logs (primary)
kubectl logs -f postgres-0 -n fashion-app

# Database logs (replica)
kubectl logs -f postgres-1 -n fashion-app
```

## Troubleshooting

### Common Issues

#### 1. Database replica not syncing

```bash
# Check replication lag
kubectl exec -it postgres-0 -n fashion-app -- psql -U postgres -c "SELECT * FROM pg_stat_replication;"

# Reinitialize replica
kubectl delete pod postgres-1 -n fashion-app
# StatefulSet will recreate and re-sync
```

#### 2. Backend can't connect to database

```bash
# Verify database is ready
kubectl get pods -n fashion-app | grep postgres

# Check database connectivity
kubectl exec -it deployment/fashion-backend -n fashion-app -- nc -zv postgres-primary 5432

# Check credentials
kubectl get secret db-secret -n fashion-app -o yaml
```

#### 3. Frontend not accessible

```bash
# Check ingress
kubectl get ingress -n fashion-app
kubectl describe ingress fashion-ingress -n fashion-app

# Verify NodePort access
curl http://<node-ip>:30080

# Check /etc/hosts entry
ping fashion.local
```

#### 4. Pods stuck in Pending

```bash
# Check PVC status
kubectl get pvc -n fashion-app

# Check storage class
kubectl get sc

# Describe pod for more details
kubectl describe pod <pod-name> -n fashion-app
```

### Emergency Procedures

#### Database Primary Failover

If postgres-0 fails:
```bash
# Promote postgres-1 to primary (manual intervention required)
kubectl exec -it postgres-1 -n fashion-app -- pg_ctl promote

# Update backend config to point to new primary
kubectl edit configmap backend-config -n fashion-app
```

#### Complete Reset

```bash
# Delete all resources
kubectl delete namespace fashion-app

# Redeploy from scratch
kubectl apply -f namespace.yaml
# ... follow deployment steps
```

## Additional Notes

- The database implements PersistentVolumeClaims with 2Gi storage allocation per replica
- Automated replica initialization is performed via pg_basebackup
- Nginx reverse proxy manages /api routing to backend services
- Frontend serves static files and routes all requests to index.html for Single Page Application routing
- All default passwords must be changed before production deployment
- Implementation of external secrets management solutions (HashiCorp Vault, Sealed Secrets) is recommended for production environments

## Security Considerations

### Production Deployment Requirements

The following security measures must be implemented before deploying to production environments:

1. Modify all default passwords in `secret.yaml` configuration
2. Implement TLS/SSL certificates for ingress endpoints
3. Enable network policies to restrict inter-pod communication
4. Configure Role-Based Access Control (RBAC) for namespace access
5. Implement container image vulnerability scanning
6. Enable pod security policies and security standards
7. Establish a comprehensive backup and disaster recovery strategy for database systems

## Additional Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [PostgreSQL Replication](https://www.postgresql.org/docs/15/high-availability.html)
- [Spring Boot Documentation](https://spring.io/projects/spring-boot)
- [Docker Compose Documentation](https://docs.docker.com/compose/)

---

**Document Version**: 1.0  
**Last Updated**: April 2026
