# Fashion Application

## Overview

The Fashion Application is a full-stack enterprise solution featuring a Spring Boot backend, modern web frontend, and PostgreSQL database with high-availability replication architecture.

## Table of Contents

- [Architecture](#architecture)
- [Key Features](#key-features)
- [Prerequisites](#prerequisites)
- [Local Development](#local-development)
- [Kubernetes Deployment](#kubernetes-deployment)
- [Configuration](#configuration)
- [Monitoring](#monitoring)
- [Additional Resources](#additional-resources)

## Architecture

### System Components

- **Frontend**: Modern web application served by Nginx
- **Backend**: Spring Boot REST API
- **Database**: PostgreSQL 15 with streaming replication for high availability
- **Reverse Proxy**: Nginx for API routing and load balancing

### Network Architecture

```
Internet → Ingress Controller → Frontend Service
                              → Backend Service
                              → Database Cluster (Primary + Replicas)
```

## Key Features

### High Availability
- Multi-replica deployment for all application layers
- PostgreSQL streaming replication with automatic failover capability
- Load-balanced read operations across database replicas
- Horizontal scaling support for frontend and backend services

### Cloud-Native Design
- Containerized microservices architecture
- Kubernetes-native deployment
- ConfigMap-based configuration management
- Secret management for sensitive data
- Health checks and readiness probes

### Performance Optimization
- Separate read and write database endpoints
- Connection pooling
- Resource limits and requests configured
- Efficient static asset delivery

## Prerequisites

### Local Development Requirements
- Docker Engine 20.10 or higher
- Docker Compose 2.0 or higher
- Minimum 4GB RAM

### Kubernetes Deployment Requirements
- Kubernetes cluster version 1.24 or higher
- kubectl command-line tool configured
- Nginx Ingress Controller
- Persistent volume provisioner
- Minimum 8GB RAM for cluster

## Local Development Environment

### Quick Start Guide

1. Clone the repository and navigate to the project directory

2. Start all services using Docker Compose:
   ```bash
   docker-compose up -d
   ```

3. Access the application:
   - Frontend: http://localhost
   - Backend API: http://localhost:8080
   - Database: localhost:5432

### Available Services

| Service | Port | Description |
|---------|------|-------------|
| Frontend | 80 | Web application interface |
| Backend | 8080 | REST API server |
| Database | 5432 | PostgreSQL database |

### Common Commands

```bash
# Start services
docker-compose up -d

# View service logs
docker-compose logs -f [service-name]

# Stop services
docker-compose down

# Rebuild services
docker-compose up -d --build
```

## Kubernetes Deployment

### Installation Procedure

1. Create namespace and configure secrets
2. Deploy database layer with StatefulSet
3. Deploy backend application
4. Deploy frontend application
5. Configure ingress for external access

### Deployment Verification

```bash
# Check pod status
kubectl get pods -n fashion-app

# Check service endpoints
kubectl get svc -n fashion-app

# Verify ingress configuration
kubectl get ingress -n fashion-app
```

### Scaling Operations

The application supports horizontal scaling for all components:

```bash
# Scale backend
kubectl scale deployment fashion-backend --replicas=<count> -n fashion-app

# Scale frontend
kubectl scale deployment fashion-frontend --replicas=<count> -n fashion-app

# Scale database replicas
kubectl scale statefulset postgres --replicas=<count> -n fashion-app
```

## Configuration

### Application Configuration

Configuration is managed through Kubernetes ConfigMaps and Secrets:

- **ConfigMaps**: Non-sensitive configuration (URLs, feature flags, profiles)
- **Secrets**: Sensitive data (credentials, API keys, certificates)

### Environment Profiles

- **Local**: Docker Compose environment for development
- **Production**: Kubernetes deployment with high availability

### Resource Allocation

Each component has defined resource requests and limits to ensure optimal performance and cluster stability.

## Monitoring

### Health Checks

All services implement health check endpoints:

- **Database**: PostgreSQL readiness probes
- **Backend**: Spring Boot Actuator health endpoints
- **Frontend**: Nginx status monitoring

### Logging

Application logs are accessible through standard Kubernetes logging:

```bash
# View application logs
kubectl logs -f deployment/<deployment-name> -n fashion-app

# View database logs
kubectl logs -f <pod-name> -n fashion-app
```

### Replication Status

Monitor database replication health:

```bash
# Connect to primary database
kubectl exec -it postgres-0 -n fashion-app -- psql -U postgres

# Check replication status
SELECT * FROM pg_stat_replication;
```

## Database Architecture

### Primary-Replica Configuration

The application uses PostgreSQL streaming replication:

- **Primary Node**: Handles all write operations
- **Replica Nodes**: Handle read operations with load balancing
- **Automatic Replication**: Real-time data synchronization

### Connection Routing

- Write operations are directed to the primary database
- Read operations are load-balanced across all replicas
- Automatic failover capabilities for high availability

## Troubleshooting

### Common Issues

#### Service Connectivity
- Verify all pods are in Running state
- Check service endpoints and networking
- Review ingress configuration

#### Database Replication
- Monitor replication lag
- Verify replica synchronization status
- Check network connectivity between database nodes

#### Resource Constraints
- Review pod resource usage
- Check persistent volume claims
- Verify storage availability

### Support Commands

```bash
# Describe pod details
kubectl describe pod <pod-name> -n fashion-app

# Check resource usage
kubectl top pods -n fashion-app

# View events
kubectl get events -n fashion-app
```

## Security Considerations

### Production Deployment

Before deploying to production:

- Configure TLS/SSL certificates for all external endpoints
- Implement network policies for pod-to-pod communication
- Enable role-based access control (RBAC)
- Implement container image scanning
- Configure automated backup procedures
- Use external secrets management solutions

## Additional Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [PostgreSQL High Availability](https://www.postgresql.org/docs/15/high-availability.html)
- [Spring Boot Documentation](https://spring.io/projects/spring-boot)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
