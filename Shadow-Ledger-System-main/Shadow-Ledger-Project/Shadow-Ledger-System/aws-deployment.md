# AWS Deployment Guide

## Architecture Overview
The Shadow Ledger System deploys on AWS using container orchestration and managed services for scalability and reliability.

## AWS Services Used

### Compute & Orchestration
- **Amazon EKS**: Kubernetes cluster for microservices
- **AWS Fargate**: Serverless container compute
- **Application Load Balancer**: Traffic distribution and SSL termination

### Data & Messaging
- **Amazon RDS PostgreSQL**: Primary database (Multi-AZ)
- **Amazon MSK**: Managed Kafka for event streaming
- **Amazon ElastiCache Redis**: Caching and session storage

### Security & Monitoring
- **AWS IAM**: Identity and access management
- **AWS Secrets Manager**: Database credentials and JWT secrets
- **Amazon CloudWatch**: Monitoring and logging
- **AWS X-Ray**: Distributed tracing

## Deployment Steps

### 1. Infrastructure Setup

#### Create VPC and Networking
```bash
aws cloudformation create-stack \
  --stack-name shadow-ledger-vpc \
  --template-body file://infrastructure/vpc.yaml \
  --parameters ParameterKey=Environment,ParameterValue=production
```

#### Setup RDS PostgreSQL
```bash
aws rds create-db-instance \
  --db-instance-identifier shadow-ledger-db \
  --db-instance-class db.t3.medium \
  --engine postgres \
  --engine-version 13.7 \
  --master-username shadowadmin \
  --master-user-password $(aws secretsmanager get-secret-value --secret-id rds-password --query SecretString --output text) \
  --allocated-storage 100 \
  --multi-az \
  --vpc-security-group-ids sg-xxxxxxxxx
```

#### Create MSK Cluster
```bash
aws kafka create-cluster \
  --cluster-name shadow-ledger-kafka \
  --broker-node-group-info file://kafka-config.json \
  --kafka-version 2.8.0
```

### 2. Container Registry

#### Build and Push Images
```bash
# Login to ECR
aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-west-2.amazonaws.com

# Build and push each service
for service in api-gateway event-service shadow-ledger-service drift-correction-service; do
  docker build -t shadow-ledger/$service ./$service/
  docker tag shadow-ledger/$service:latest 123456789012.dkr.ecr.us-west-2.amazonaws.com/shadow-ledger/$service:latest
  docker push 123456789012.dkr.ecr.us-west-2.amazonaws.com/shadow-ledger/$service:latest
done
```

### 3. EKS Cluster Setup

#### Create EKS Cluster
```bash
eksctl create cluster \
  --name shadow-ledger-cluster \
  --region us-west-2 \
  --version 1.24 \
  --nodegroup-name shadow-ledger-nodes \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 6 \
  --managed
```

#### Deploy Services
```bash
# Apply Kubernetes manifests
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secrets.yaml
kubectl apply -f k8s/services/
kubectl apply -f k8s/deployments/
kubectl apply -f k8s/ingress.yaml
```

### 4. Configuration Management

#### Environment Variables
```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: shadow-ledger-config
data:
  SPRING_PROFILES_ACTIVE: "aws"
  KAFKA_BOOTSTRAP_SERVERS: "shadow-ledger-kafka.amazonaws.com:9092"
  POSTGRES_HOST: "shadow-ledger-db.cluster-xxx.us-west-2.rds.amazonaws.com"
  REDIS_HOST: "shadow-ledger-cache.xxx.cache.amazonaws.com"
```

#### Secrets Management
```bash
# Store sensitive data in Secrets Manager
aws secretsmanager create-secret \
  --name shadow-ledger/jwt-secret \
  --secret-string "your-jwt-secret-key"

aws secretsmanager create-secret \
  --name shadow-ledger/db-credentials \
  --secret-string '{"username":"shadowadmin","password":"your-db-password"}'
```

## Scaling Configuration

### Horizontal Pod Autoscaler
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: shadow-ledger-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: shadow-ledger-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### Database Scaling
- **Read Replicas**: Create for read-heavy workloads
- **Connection Pooling**: Configure PgBouncer for connection management
- **Partitioning**: Implement table partitioning for large datasets

## Monitoring & Alerts

### CloudWatch Alarms
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "ShadowLedger-HighErrorRate" \
  --alarm-description "High error rate detected" \
  --metric-name "ErrorRate" \
  --namespace "ShadowLedger" \
  --statistic "Average" \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 5.0 \
  --comparison-operator "GreaterThanThreshold" \
  --alarm-actions "arn:aws:sns:us-west-2:123456789012:shadow-ledger-alerts"
```

### X-Ray Tracing
Enable distributed tracing for request flow analysis:
```yaml
env:
- name: AWS_XRAY_TRACING_NAME
  value: "ShadowLedgerService"
- name: _X_AMZN_TRACE_ID
  valueFrom:
    fieldRef:
      fieldPath: metadata.annotations['xray.amazonaws.com/trace-id']
```

## Cost Optimization
- Use Spot Instances for non-critical workloads
- Implement lifecycle policies for log retention
- Schedule non-production environments to reduce costs
- Monitor unused resources with AWS Cost Explorer

## Backup & Disaster Recovery
- **RDS Automated Backups**: 7-day retention
- **Cross-Region Replication**: For disaster recovery
- **Kafka Topic Backups**: Regular snapshots to S3
- **Application State**: Stateless design enables quick recovery
