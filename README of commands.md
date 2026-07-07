# Prerequisites

## System requirements

```bash
# Recommended OS
Ubuntu 22.04 / Ubuntu 24.04 
```

```bash
# Required local access
AWS account access
EKS cluster access
Docker permission on local machine
Private ECR access
```

## Required tools

```text
aws cli
kubectl
helm
docker
git
curl
jq
tar
openssl
```

## Install common Linux packages

```bash
sudo apt update
```

```bash
sudo apt install -y \
  curl \
  wget \
  unzip \
  tar \
  git \
  jq \
  ca-certificates \
  gnupg \
  lsb-release \
  openssl
```

## Install AWS CLI v2

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" \
  -o "awscliv2.zip"
```

```bash
unzip awscliv2.zip
```

```bash
sudo ./aws/install
```

```bash
aws --version
```

## Configure AWS CLI

```bash
aws configure
```

```bash
aws sts get-caller-identity
```

## Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

```bash
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

```bash
kubectl version --client
```

## Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

```bash
helm version
```

## Install Docker

```bash
sudo apt update
```

```bash
sudo apt install -y docker.io
```

```bash
sudo systemctl enable docker
```

```bash
sudo systemctl start docker
```

```bash
sudo usermod -aG docker $USER
```

```bash
newgrp docker
```

```bash
docker version
```

## Login to AWS ECR

```bash
export REGION=ap-south-1
export ACCOUNT_ID=454143665149
export ECR_REGISTRY=$ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com
```

```bash
aws ecr get-login-password --region $REGION \
  | docker login --username AWS --password-stdin $ECR_REGISTRY
```

## Clone or open project repository

```bash
cd ~
```

```bash
git clone <YOUR_REPOSITORY_URL> retail-store-sample-app
```

```bash
cd ~/retail-store-sample-app
```

## Required AWS permissions

```text
EKS cluster read/write access
EC2 VPC endpoint read/write access
ECR repository create/read/write access
IAM role/policy attach access
S3 bucket access for Mimir and Tempo
EKS Pod Identity association access
```

## Required AWS resources before setup

```text
Existing EKS cluster
Existing private subnets
Existing node role
Existing ECR repositories or permission to create them
Existing S3 buckets for Mimir and Tempo
S3 Gateway VPC Endpoint
ECR API VPC Endpoint
ECR Docker VPC Endpoint
EKS Auth VPC Endpoint
```

## Required S3 buckets

```text
retail-mimir-blocks-454143665149
retail-mimir-ruler-454143665149
retail-tempo-traces-454143665149
```

## Required IAM roles and policies

```text
AmazonEKSAutoNodeRole
ObservabilityS3Policy
EKSAuthAssumeRoleForPodIdentity
MimirS3PodRole
TempoS3PodRole
```

## Required Kubernetes namespaces

```text
monitoring
logging
elastic-system
application
```

## Important note for private EKS cluster

```text
All container images must be available in private ECR.
Worker node subnet route tables must be attached to S3 Gateway Endpoint.
Without this, pods may fail with ImagePullBackOff or ErrImagePull.
```

---

# EKS Observability Setup Commands

## 0. Start

```bash
cd ~/retail-store-sample-app

export AWS_PAGER=""
export REGION=ap-south-1
export ACCOUNT_ID=454143665149
export CLUSTER_NAME=retail-demo-app
export ECR_REGISTRY=$ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com
```

```bash
aws sts get-caller-identity

aws eks update-kubeconfig   --region $REGION   --name $CLUSTER_NAME

kubectl get nodes
kubectl get pods -A
```

## 1. ECR Login

```bash
aws ecr get-login-password --region $REGION   | docker login --username AWS --password-stdin $ECR_REGISTRY
```

## 2. Create ECR Repositories

```bash
aws ecr create-repository --repository-name observability/mimir --region $REGION || true
aws ecr create-repository --repository-name observability/alloy --region $REGION || true
aws ecr create-repository --repository-name observability/kube-state-metrics --region $REGION || true
aws ecr create-repository --repository-name observability/nginx-unprivileged --region $REGION || true
aws ecr create-repository --repository-name observability/prometheus-config-reloader --region $REGION || true
aws ecr create-repository --repository-name observability/fluent-bit --region $REGION || true
aws ecr create-repository --repository-name observability/tempo --region $REGION || true
aws ecr create-repository --repository-name observability/telemetrygen --region $REGION || true
aws ecr create-repository --repository-name observability/elasticsearch --region $REGION || true
aws ecr create-repository --repository-name observability/kibana --region $REGION || true
aws ecr create-repository --repository-name observability/eck-operator --region $REGION || true
aws ecr create-repository --repository-name observability/grafana --region $REGION || true
aws ecr create-repository --repository-name observability/k8s-sidecar --region $REGION || true
aws ecr create-repository --repository-name observability/amazonlinux --region $REGION || true
```

## 3. Mirror Images to ECR

```bash
docker pull grafana/mimir:latest
docker tag grafana/mimir:latest $ECR_REGISTRY/observability/mimir:latest
docker push $ECR_REGISTRY/observability/mimir:latest
```

```bash
docker pull grafana/alloy:latest
docker tag grafana/alloy:latest $ECR_REGISTRY/observability/alloy:latest
docker push $ECR_REGISTRY/observability/alloy:latest
```

```bash
docker pull registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.17.0
docker tag registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.17.0 $ECR_REGISTRY/observability/kube-state-metrics:v2.17.0
docker push $ECR_REGISTRY/observability/kube-state-metrics:v2.17.0
```

```bash
docker pull nginxinc/nginx-unprivileged:1.29-alpine
docker tag nginxinc/nginx-unprivileged:1.29-alpine $ECR_REGISTRY/observability/nginx-unprivileged:1.29-alpine
docker push $ECR_REGISTRY/observability/nginx-unprivileged:1.29-alpine
```

```bash
docker pull quay.io/prometheus-operator/prometheus-config-reloader:v0.91.0
docker tag quay.io/prometheus-operator/prometheus-config-reloader:v0.91.0 $ECR_REGISTRY/observability/prometheus-config-reloader:v0.91.0
docker push $ECR_REGISTRY/observability/prometheus-config-reloader:v0.91.0
```

```bash
docker pull fluent/fluent-bit:3.2.10
docker tag fluent/fluent-bit:3.2.10 $ECR_REGISTRY/observability/fluent-bit:3.2.10
docker push $ECR_REGISTRY/observability/fluent-bit:3.2.10
```

```bash
docker pull grafana/tempo:2.6.1
docker tag grafana/tempo:2.6.1 $ECR_REGISTRY/observability/tempo:2.6.1
docker push $ECR_REGISTRY/observability/tempo:2.6.1
```

```bash
docker pull ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:v0.129.0
docker tag ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:v0.129.0 $ECR_REGISTRY/observability/telemetrygen:v0.129.0
docker push $ECR_REGISTRY/observability/telemetrygen:v0.129.0
```

```bash
docker pull docker.elastic.co/elasticsearch/elasticsearch:8.15.3
docker tag docker.elastic.co/elasticsearch/elasticsearch:8.15.3 $ECR_REGISTRY/observability/elasticsearch:8.15.3
docker push $ECR_REGISTRY/observability/elasticsearch:8.15.3
```

```bash
docker pull docker.elastic.co/kibana/kibana:8.15.3
docker tag docker.elastic.co/kibana/kibana:8.15.3 $ECR_REGISTRY/observability/kibana:8.15.3
docker push $ECR_REGISTRY/observability/kibana:8.15.3
```

```bash
docker pull docker.elastic.co/eck/eck-operator:2.15.0
docker tag docker.elastic.co/eck/eck-operator:2.15.0 $ECR_REGISTRY/observability/eck-operator:2.15.0
docker push $ECR_REGISTRY/observability/eck-operator:2.15.0
```

```bash
docker pull grafana/grafana:latest
docker tag grafana/grafana:latest $ECR_REGISTRY/observability/grafana:latest
docker push $ECR_REGISTRY/observability/grafana:latest
```

```bash
docker pull quay.io/kiwigrid/k8s-sidecar:1.30.10
docker tag quay.io/kiwigrid/k8s-sidecar:1.30.10 $ECR_REGISTRY/observability/k8s-sidecar:1.30.10
docker push $ECR_REGISTRY/observability/k8s-sidecar:1.30.10
```

```bash
docker pull public.ecr.aws/amazonlinux/amazonlinux:2023
docker tag public.ecr.aws/amazonlinux/amazonlinux:2023 $ECR_REGISTRY/observability/amazonlinux:2023
docker push $ECR_REGISTRY/observability/amazonlinux:2023
```

## 4. IAM Policy Checks

```bash
aws iam list-attached-role-policies   --role-name AmazonEKSAutoNodeRole   --query 'AttachedPolicies[*].PolicyArn'   --output text
```

```bash
aws iam attach-role-policy   --role-name AmazonEKSAutoNodeRole   --policy-arn arn:aws:iam::454143665149:policy/ObservabilityS3Policy || true
```

```bash
aws iam attach-role-policy   --role-name AmazonEKSAutoNodeRole   --policy-arn arn:aws:iam::454143665149:policy/EKSAuthAssumeRoleForPodIdentity || true
```

## 5. VPC Endpoint Checks

```bash
aws ec2 describe-vpc-endpoints   --region $REGION   --vpc-endpoint-ids vpce-0d4e2802a27dc4964   --query 'VpcEndpoints[0].RouteTableIds'   --output table
```

```bash
NODE_IDS=$(kubectl get nodes -o jsonpath='{range .items[*]}{.spec.providerID}{"\n"}{end}' | sed 's|aws:///[^/]*/||')
echo $NODE_IDS
```

```bash
aws ec2 describe-instances   --instance-ids $NODE_IDS   --region $REGION   --query 'Reservations[*].Instances[*].[InstanceId,SubnetId,PrivateIpAddress,SecurityGroups[*].GroupId]'   --output table
```

```bash
aws ec2 describe-route-tables   --region $REGION   --filters "Name=association.subnet-id,Values=<SUBNET_ID>"   --query 'RouteTables[*].RouteTableId'   --output text
```

```bash
aws ec2 modify-vpc-endpoint   --vpc-endpoint-id vpce-0d4e2802a27dc4964   --add-route-table-ids <ROUTE_TABLE_ID>   --region $REGION
```

```bash
aws ec2 describe-vpc-endpoints   --region $REGION   --filters "Name=service-name,Values=com.amazonaws.ap-south-1.eks-auth"   --query 'VpcEndpoints[*].[VpcEndpointId,State,PrivateDnsEnabled]'   --output table
```

## 6. Namespaces and StorageClass

```bash
kubectl apply -f k8s/observability/00-namespaces.yaml
kubectl apply -f k8s/observability/gp3-storageclass.yaml
kubectl get ns
kubectl get storageclass
```

## 7. Helm Repos

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add elastic https://helm.elastic.co
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update
```

## 8. kube-state-metrics

```bash
helm upgrade --install kube-state-metrics prometheus-community/kube-state-metrics   -n monitoring   -f k8s/observability/kube-state-metrics/kube-state-metrics-values.yaml
```

```bash
kubectl get pods -n monitoring | grep kube-state
```

## 9. Mimir

```bash
helm upgrade --install mimir grafana/mimir-distributed   -n monitoring   -f k8s/observability/mimir/mimir-values.yaml
```

```bash
aws eks create-pod-identity-association   --cluster-name $CLUSTER_NAME   --namespace monitoring   --service-account mimir   --role-arn arn:aws:iam::454143665149:role/MimirS3PodRole   --region $REGION || true
```

```bash
kubectl delete pod -n monitoring -l app.kubernetes.io/instance=mimir
```

```bash
kubectl get pods -n monitoring | grep mimir
```

## 10. Alloy

```bash
kubectl apply -f k8s/observability/alloy/alloy-extra-rbac.yaml
```

```bash
helm upgrade --install alloy grafana/alloy   -n monitoring   -f k8s/observability/alloy/alloy-values.yaml
```

```bash
kubectl -n monitoring set image daemonset/alloy   config-reloader=$ECR_REGISTRY/observability/prometheus-config-reloader:v0.91.0
```

```bash
kubectl rollout status daemonset/alloy -n monitoring
kubectl get pods -n monitoring | grep alloy
```

## 11. ECK Operator

```bash
helm upgrade --install elastic-operator elastic/eck-operator   --version 2.15.0   -n elastic-system   --create-namespace   --set image.repository=$ECR_REGISTRY/observability/eck-operator   --set image.tag=2.15.0   --set image.pullPolicy=IfNotPresent
```

```bash
kubectl get pods -n elastic-system
```

```bash
kubectl get pod elastic-operator-0 -n elastic-system   -o jsonpath='{.spec.containers[0].image}{"\n"}'
```

## 12. Elasticsearch and Kibana

```bash
kubectl apply -f k8s/observability/elasticsearch/elasticsearch-kibana.yaml
```

```bash
kubectl get pods -n logging
kubectl get elasticsearch -n logging
kubectl get kibana -n logging
kubectl get pods -n logging -w
```

## 13. Elasticsearch Secret Copy

```bash
ES_PASS=$(kubectl -n logging get secret retail-logs-es-elastic-user -o jsonpath='{.data.elastic}' | base64 -d)
```

```bash
kubectl -n monitoring create secret generic retail-logs-es-elastic-user   --from-literal=elastic="$ES_PASS"   --dry-run=client -o yaml | kubectl apply -f -
```

```bash
kubectl get secret retail-logs-es-elastic-user -n monitoring
```

## 14. Fluent Bit

```bash
helm upgrade --install fluent-bit fluent/fluent-bit   -n logging   -f k8s/observability/fluent-bit/fluent-bit-values.yaml
```

```bash
kubectl get pods -n logging | grep fluent-bit
```

```bash
kubectl logs -n logging -l app.kubernetes.io/name=fluent-bit --tail=100
```

## 15. Tempo

```bash
helm upgrade --install tempo grafana/tempo   -n monitoring   -f k8s/observability/tempo/tempo-values.yaml
```

```bash
aws eks create-pod-identity-association   --cluster-name $CLUSTER_NAME   --namespace monitoring   --service-account tempo   --role-arn arn:aws:iam::454143665149:role/TempoS3PodRole   --region $REGION || true
```

```bash
kubectl delete pod tempo-0 -n monitoring --ignore-not-found
```

```bash
kubectl get pods -n monitoring | grep tempo
```

```bash
kubectl get svc tempo -n monitoring -o yaml | grep -E "3200|4317|4318"
```

## 16. Telemetrygen CronJobs

```bash
kubectl apply -f k8s/observability/tempo/telemetrygen-multi-services-cronjob.yaml
```

```bash
kubectl get cronjob -n monitoring | grep telemetrygen
kubectl get jobs -n monitoring | grep telemetrygen
kubectl get pods -n monitoring | grep telemetrygen
```

```bash
kubectl create job telemetrygen-retail-manual --from=cronjob/telemetrygen-retail-cron -n monitoring || true
kubectl create job telemetrygen-cart-manual --from=cronjob/telemetrygen-cart-cron -n monitoring || true
kubectl create job telemetrygen-payment-manual --from=cronjob/telemetrygen-payment-cron -n monitoring || true
kubectl create job telemetrygen-order-manual --from=cronjob/telemetrygen-order-cron -n monitoring || true
kubectl create job telemetrygen-inventory-manual --from=cronjob/telemetrygen-inventory-cron -n monitoring || true
```

```bash
kubectl logs job/telemetrygen-cart-manual -n monitoring
```

## 17. Grafana Dashboard ConfigMap

```bash
kubectl apply -f k8s/observability/grafana/grafana-dashboards-configmap.yaml
```

```bash
kubectl label configmap grafana-dashboards   -n monitoring   grafana_dashboard=1   --overwrite
```

```bash
kubectl annotate configmap grafana-dashboards   -n monitoring   grafana_folder=Observability   --overwrite
```

```bash
kubectl get configmap grafana-dashboards -n monitoring
```

## 18. Grafana

```bash
helm upgrade --install grafana grafana/grafana   -n monitoring   -f k8s/observability/grafana/grafana-values.yaml
```

```bash
kubectl get pods -n monitoring | grep grafana
```

```bash
kubectl get pod -n monitoring -l app.kubernetes.io/name=grafana   -o jsonpath='{.items[0].spec.containers[*].name}{"\n"}'
```

```bash
kubectl delete pod -n monitoring -l app.kubernetes.io/name=grafana
```

```bash
kubectl get pods -n monitoring | grep grafana
```

```bash
kubectl -n monitoring port-forward svc/grafana 3000:80
```

```bash
echo "Grafana URL: http://localhost:3000"
echo "Username: admin"
echo "Password: admin123"
```

## 19. Mimir Validation

```bash
kubectl -n monitoring port-forward svc/mimir-gateway 9009:80
```

```bash
curl -G http://localhost:9009/prometheus/api/v1/query   --data-urlencode 'query=up'
```

```bash
curl -G http://localhost:9009/prometheus/api/v1/query   --data-urlencode 'query=kube_pod_status_phase'
```

```bash
curl -G http://localhost:9009/prometheus/api/v1/query   --data-urlencode 'query=sum(kube_pod_status_phase{phase="Running"})'
```

```bash
curl -G http://localhost:9009/prometheus/api/v1/query   --data-urlencode 'query=up{job="cadvisor"}'
```

```bash
curl -G http://localhost:9009/prometheus/api/v1/query   --data-urlencode 'query=container_cpu_usage_seconds_total'
```

```bash
curl -G http://localhost:9009/prometheus/api/v1/query   --data-urlencode 'query=container_memory_working_set_bytes'
```

```bash
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
echo $NODE
```

```bash
kubectl get --raw "/api/v1/nodes/$NODE/proxy/metrics/cadvisor"   | grep container_cpu_usage_seconds_total | head
```

```bash
kubectl auth can-i get nodes/proxy   --as=system:serviceaccount:monitoring:alloy
```

```bash
kubectl auth can-i list nodes   --as=system:serviceaccount:monitoring:alloy
```

## 20. Elasticsearch Validation

```bash
ES_PASS=$(kubectl -n logging get secret retail-logs-es-elastic-user -o jsonpath='{.data.elastic}' | base64 -d)
```

```bash
kubectl -n logging port-forward svc/retail-logs-es-http 9200:9200
```

```bash
curl -k -u elastic:$ES_PASS https://localhost:9200/_cat/indices?v
```

```bash
curl -k -u elastic:$ES_PASS https://localhost:9200/_cat/indices?v | grep fluentbit
```

```bash
curl -k -u elastic:$ES_PASS   "https://localhost:9200/fluentbit-*/_search?size=1&pretty"
```

```bash
curl -k -u elastic:$ES_PASS   "https://localhost:9200/fluentbit-*/_search?size=5&pretty"   | grep -E '"pod_name"|"namespace_name"|"container_name"|"log"'
```

## 21. Tempo Validation

```bash
kubectl -n monitoring port-forward svc/tempo 3200:3200
```

```bash
curl http://localhost:3200/ready
```

```bash
curl -G "http://localhost:3200/api/search"   --data-urlencode 'q={}'   --data-urlencode 'limit=20'
```

```bash
START=$(date -u -d '2 hours ago' +%s)
END=$(date -u +%s)
```

```bash
curl -G "http://localhost:3200/api/search"   --data-urlencode 'q={}'   --data-urlencode "start=$START"   --data-urlencode "end=$END"   --data-urlencode 'limit=20'
```

```bash
curl -s http://localhost:3200/metrics | grep -E "tempo_distributor_spans_received_total|tempo_discarded_spans_total|tempo_receiver"
```

## 22. Dashboard Backup Commands

```bash
mkdir -p k8s/observability/grafana/dashboards
```

```bash
curl -s -u admin:admin123   http://localhost:3000/api/dashboards/uid/adpthm8   | jq '.dashboard | .id=null | .version=0'   > k8s/observability/grafana/dashboards/metrics-dashboard.json
```

```bash
curl -s -u admin:admin123   http://localhost:3000/api/dashboards/uid/adwh54l   | jq '.dashboard | .id=null | .version=0'   > k8s/observability/grafana/dashboards/logs-dashboard.json
```

```bash
curl -s -u admin:admin123   http://localhost:3000/api/dashboards/uid/adjzjts   | jq '.dashboard | .id=null | .version=0'   > k8s/observability/grafana/dashboards/traces-dashboard.json
```

```bash
ls -lh k8s/observability/grafana/dashboards/
```

```bash
kubectl create configmap grafana-dashboards   -n monitoring   --from-file=k8s/observability/grafana/dashboards/   --dry-run=client -o yaml   > k8s/observability/grafana/grafana-dashboards-configmap.yaml
```

```bash
python3 - <<'PY'
from pathlib import Path

p = Path("k8s/observability/grafana/grafana-dashboards-configmap.yaml")
s = p.read_text()

insert = '''metadata:
  labels:
    grafana_dashboard: "1"
  annotations:
    grafana_folder: Observability
'''

if "metadata:\n" in s and "grafana_dashboard" not in s:
    s = s.replace("metadata:\n", insert, 1)

p.write_text(s)
PY
```

```bash
kubectl apply -f k8s/observability/grafana/grafana-dashboards-configmap.yaml
```

```bash
kubectl delete pod -n monitoring -l app.kubernetes.io/name=grafana
```

## 23. Troubleshooting Commands

```bash
kubectl get pods -A
```

```bash
kubectl get events -A --sort-by=.lastTimestamp
```

```bash
kubectl describe pod <POD_NAME> -n <NAMESPACE>
```

```bash
kubectl logs <POD_NAME> -n <NAMESPACE> --tail=100
```

```bash
kubectl get pod <POD_NAME> -n <NAMESPACE> -o wide
```

```bash
kubectl get pod <POD_NAME> -n <NAMESPACE>   -o jsonpath='{range .spec.initContainers[*]}{.name}{" -> "}{.image}{"\n"}{end}{range .spec.containers[*]}{.name}{" -> "}{.image}{"\n"}{end}'
```

```bash
kubectl describe pod <POD_NAME> -n <NAMESPACE> | grep -A8 -B5 "Failed"
```

```bash
aws ecr get-login-password --region $REGION   | docker login --username AWS --password-stdin $ECR_REGISTRY
```

```bash
docker push $ECR_REGISTRY/<REPOSITORY>:<TAG>
```

```bash
aws ecr describe-images   --repository-name <REPOSITORY>   --image-ids imageTag=<TAG>   --region $REGION
```

## 24. Final Status Checks

```bash
kubectl get pods -n monitoring | egrep 'grafana|mimir|alloy|tempo|kube-state|telemetrygen'
```

```bash
kubectl get pods -n logging | egrep 'fluent-bit|retail-logs|kibana'
```

```bash
kubectl get pods -n elastic-system
```

```bash
kubectl get svc -n monitoring
```

```bash
kubectl get svc -n logging
```

```bash
kubectl get cronjob -n monitoring | grep telemetrygen
```

```bash
kubectl get elasticsearch -n logging
```

```bash
kubectl get kibana -n logging
```
