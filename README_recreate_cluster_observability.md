# Recreate EKS Observability Stack After Cluster Deletion

This restores:

```text
Private EKS Cluster
  -> Metrics: Alloy + kube-state-metrics -> Mimir -> Grafana
  -> Logs: Fluent Bit -> Elasticsearch/Kibana -> Grafana
  -> Traces: Tempo -> Grafana
  -> Dashboards: ConfigMap + Grafana sidecar
```

---

## 0. Before Deleting Current Cluster

### Save dashboards from current Grafana

Open port-forward:

```bash
kubectl -n monitoring port-forward svc/grafana 3000:80
```

In another terminal:

```bash
cd ~/retail-store-sample-app
mkdir -p k8s/observability/grafana/dashboards
```

Get dashboard UIDs:

```bash
curl -s -u admin:admin123 \
  "http://localhost:3000/api/search?type=dash-db" \
  | jq -r '.[] | "\(.title) -> \(.uid)"'
```

Export dashboards using actual UIDs:

```bash
curl -s -u admin:admin123 \
  http://localhost:3000/api/dashboards/uid/<METRICS_UID> \
  | jq '.dashboard | .id=null | .version=0' \
  > k8s/observability/grafana/dashboards/metrics-dashboard.json
```

```bash
curl -s -u admin:admin123 \
  http://localhost:3000/api/dashboards/uid/<LOGS_UID> \
  | jq '.dashboard | .id=null | .version=0' \
  > k8s/observability/grafana/dashboards/logs-dashboard.json
```

```bash
curl -s -u admin:admin123 \
  http://localhost:3000/api/dashboards/uid/<TRACES_UID> \
  | jq '.dashboard | .id=null | .version=0' \
  > k8s/observability/grafana/dashboards/traces-dashboard.json
```

Validate dashboard JSON files:

```bash
ls -lh k8s/observability/grafana/dashboards/
wc -c k8s/observability/grafana/dashboards/*.json
jq '.title, .uid' k8s/observability/grafana/dashboards/*.json
```

If any file is `0` bytes or `jq` gives error, export again before deleting the cluster.

---

## 1. Create Dashboard ConfigMap File Before Deletion

```bash
kubectl create configmap grafana-dashboards \
  -n monitoring \
  --from-file=k8s/observability/grafana/dashboards/ \
  --dry-run=client -o yaml \
  > k8s/observability/grafana/grafana-dashboards-configmap.yaml
```

Add Grafana sidecar label and folder annotation:

```bash
python3 - <<'PY'
from pathlib import Path

p = Path("k8s/observability/grafana/grafana-dashboards-configmap.yaml")
s = p.read_text()

if "grafana_dashboard" not in s:
    s = s.replace(
        "metadata:\n",
        """metadata:
  labels:
    grafana_dashboard: "1"
  annotations:
    grafana_folder: Observability
""",
        1
    )

p.write_text(s)
PY
```

Check:

```bash
grep -A8 "metadata:" k8s/observability/grafana/grafana-dashboards-configmap.yaml
```

---

## 2. Commit Final Files Before Deleting Cluster

```bash
git status
```

```bash
git add .
```

```bash
git commit -m "Save final observability setup and dashboards"
```

```bash
git push
```

---

## 3. Do Not Delete These AWS Resources

Keep these resources when deleting the cluster:

```text
ECR repositories
S3 buckets
IAM roles
IAM policies
VPC
Private subnets
VPC endpoints
```

Important S3 buckets:

```text
retail-mimir-blocks-454143665149
retail-mimir-ruler-454143665149
retail-tempo-traces-454143665149
```

Important IAM resources:

```text
AmazonEKSAutoNodeRole
ObservabilityS3Policy
EKSAuthAssumeRoleForPodIdentity
MimirS3PodRole
TempoS3PodRole
```

Important VPC endpoint:

```text
S3 Gateway Endpoint: vpce-0d4e2802a27dc4964
```

---

# Tomorrow Steps After New Cluster Is Created

## 4. Go to Project Folder

```bash
cd ~/retail-store-sample-app
```

```bash
git pull
```

---

## 5. Set Variables

```bash
export AWS_PAGER=""
export REGION=ap-south-1
export ACCOUNT_ID=454143665149
export CLUSTER_NAME=retail-demo-app
export ECR_REGISTRY=$ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com
export VPC_ID=vpc-02358ddc1cb955bcd
export S3_ENDPOINT_ID=vpce-0d4e2802a27dc4964
```

---

## 6. Verify AWS Account

```bash
aws sts get-caller-identity
```

Expected account:

```text
454143665149
```

---

## 7. Connect kubectl to New Cluster

```bash
aws eks update-kubeconfig \
  --region $REGION \
  --name $CLUSTER_NAME
```

```bash
kubectl get nodes
```

```bash
kubectl get pods -A
```

---

## 8. Login to ECR

```bash
aws ecr get-login-password --region $REGION \
  | docker login --username AWS --password-stdin $ECR_REGISTRY
```

---

## 9. Check Required ECR Images

```bash
aws ecr describe-repositories --region $REGION \
  --query 'repositories[*].repositoryName' \
  --output text
```

Check important images:

```bash
aws ecr describe-images --repository-name observability/grafana --region $REGION --query 'imageDetails[*].imageTags' --output table
aws ecr describe-images --repository-name observability/k8s-sidecar --region $REGION --query 'imageDetails[*].imageTags' --output table
aws ecr describe-images --repository-name observability/mimir --region $REGION --query 'imageDetails[*].imageTags' --output table
aws ecr describe-images --repository-name observability/alloy --region $REGION --query 'imageDetails[*].imageTags' --output table
aws ecr describe-images --repository-name observability/fluent-bit --region $REGION --query 'imageDetails[*].imageTags' --output table
aws ecr describe-images --repository-name observability/tempo --region $REGION --query 'imageDetails[*].imageTags' --output table
aws ecr describe-images --repository-name observability/elasticsearch --region $REGION --query 'imageDetails[*].imageTags' --output table
aws ecr describe-images --repository-name observability/kibana --region $REGION --query 'imageDetails[*].imageTags' --output table
aws ecr describe-images --repository-name observability/eck-operator --region $REGION --query 'imageDetails[*].imageTags' --output table
```

If any image is missing, push it again before installing that component.

---

## 10. Check Node Role Policies

```bash
aws iam list-attached-role-policies \
  --role-name AmazonEKSAutoNodeRole \
  --query 'AttachedPolicies[*].PolicyArn' \
  --output text
```

Attach required policies if missing:

```bash
aws iam attach-role-policy \
  --role-name AmazonEKSAutoNodeRole \
  --policy-arn arn:aws:iam::454143665149:policy/ObservabilityS3Policy || true
```

```bash
aws iam attach-role-policy \
  --role-name AmazonEKSAutoNodeRole \
  --policy-arn arn:aws:iam::454143665149:policy/EKSAuthAssumeRoleForPodIdentity || true
```

---

## 11. Check S3 Gateway Endpoint Route Tables

Private EKS needs S3 Gateway Endpoint route tables for ECR image layer downloads.

Get node instance IDs:

```bash
NODE_IDS=$(kubectl get nodes -o jsonpath='{range .items[*]}{.spec.providerID}{"\n"}{end}' | sed 's|aws:///[^/]*/||')
echo $NODE_IDS
```

Get node subnets:

```bash
aws ec2 describe-instances \
  --instance-ids $NODE_IDS \
  --region $REGION \
  --query 'Reservations[*].Instances[*].[InstanceId,SubnetId,PrivateIpAddress]' \
  --output table
```

Check S3 endpoint route tables:

```bash
aws ec2 describe-vpc-endpoints \
  --region $REGION \
  --vpc-endpoint-ids $S3_ENDPOINT_ID \
  --query 'VpcEndpoints[0].RouteTableIds' \
  --output table
```

Find route table for node subnet:

```bash
aws ec2 describe-route-tables \
  --region $REGION \
  --filters "Name=association.subnet-id,Values=<SUBNET_ID>" \
  --query 'RouteTables[*].RouteTableId' \
  --output text
```

Attach missing route table to S3 endpoint:

```bash
aws ec2 modify-vpc-endpoint \
  --vpc-endpoint-id $S3_ENDPOINT_ID \
  --add-route-table-ids <ROUTE_TABLE_ID> \
  --region $REGION
```

Repeat for every new node subnet route table.

---

## 12. Check EKS Auth VPC Endpoint

```bash
aws ec2 describe-vpc-endpoints \
  --region $REGION \
  --filters "Name=service-name,Values=com.amazonaws.ap-south-1.eks-auth" \
  --query 'VpcEndpoints[*].[VpcEndpointId,State,PrivateDnsEnabled]' \
  --output table
```

Expected:

```text
State = available
PrivateDnsEnabled = True
```

---

## 13. Apply Namespaces and StorageClass

```bash
kubectl apply -f k8s/observability/00-namespaces.yaml
```

```bash
kubectl apply -f k8s/observability/gp3-storageclass.yaml
```

```bash
kubectl get ns
```

```bash
kubectl get storageclass
```

---

## 14. Add Helm Repositories

```bash
helm repo add grafana https://grafana.github.io/helm-charts
```

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

```bash
helm repo add elastic https://helm.elastic.co
```

```bash
helm repo add fluent https://fluent.github.io/helm-charts
```

```bash
helm repo update
```

---

# Metrics Stack

## 15. Install kube-state-metrics

```bash
helm upgrade --install kube-state-metrics prometheus-community/kube-state-metrics \
  -n monitoring \
  -f k8s/observability/kube-state-metrics/kube-state-metrics-values.yaml
```

```bash
kubectl get pods -n monitoring | grep kube-state
```

---

## 16. Install Mimir

```bash
helm upgrade --install mimir grafana/mimir-distributed \
  -n monitoring \
  -f k8s/observability/mimir/mimir-values.yaml
```

Create Pod Identity association:

```bash
aws eks create-pod-identity-association \
  --cluster-name $CLUSTER_NAME \
  --namespace monitoring \
  --service-account mimir \
  --role-arn arn:aws:iam::454143665149:role/MimirS3PodRole \
  --region $REGION || true
```

Restart Mimir:

```bash
kubectl delete pod -n monitoring -l app.kubernetes.io/instance=mimir
```

```bash
kubectl get pods -n monitoring | grep mimir
```

---

## 17. Install Alloy

```bash
kubectl apply -f k8s/observability/alloy/alloy-extra-rbac.yaml
```

```bash
helm upgrade --install alloy grafana/alloy \
  -n monitoring \
  -f k8s/observability/alloy/alloy-values.yaml
```

```bash
kubectl -n monitoring set image daemonset/alloy \
  config-reloader=$ECR_REGISTRY/observability/prometheus-config-reloader:v0.91.0
```

```bash
kubectl rollout status daemonset/alloy -n monitoring
```

```bash
kubectl get pods -n monitoring | grep alloy
```

---

# Logs Stack

## 18. Install ECK Operator

```bash
helm upgrade --install elastic-operator elastic/eck-operator \
  --version 2.15.0 \
  -n elastic-system \
  --create-namespace \
  --set image.repository=$ECR_REGISTRY/observability/eck-operator \
  --set image.tag=2.15.0 \
  --set image.pullPolicy=IfNotPresent
```

```bash
kubectl get pods -n elastic-system
```

---

## 19. Install Elasticsearch and Kibana

```bash
kubectl apply -f k8s/observability/elasticsearch/elasticsearch-kibana.yaml
```

```bash
kubectl get pods -n logging
```

```bash
kubectl get elasticsearch -n logging
```

```bash
kubectl get kibana -n logging
```

Wait until ready:

```bash
kubectl get pods -n logging -w
```

---

## 20. Copy Elasticsearch Secret to Monitoring Namespace

```bash
ES_PASS=$(kubectl -n logging get secret retail-logs-es-elastic-user -o jsonpath='{.data.elastic}' | base64 -d)
```

```bash
kubectl -n monitoring create secret generic retail-logs-es-elastic-user \
  --from-literal=elastic="$ES_PASS" \
  --dry-run=client -o yaml | kubectl apply -f -
```

```bash
kubectl get secret retail-logs-es-elastic-user -n monitoring
```

---

## 21. Install Fluent Bit

```bash
helm upgrade --install fluent-bit fluent/fluent-bit \
  -n logging \
  -f k8s/observability/fluent-bit/fluent-bit-values.yaml
```

```bash
kubectl get pods -n logging | grep fluent-bit
```

```bash
kubectl logs -n logging -l app.kubernetes.io/name=fluent-bit --tail=100
```

---

# Tracing Stack

## 22. Install Tempo

```bash
helm upgrade --install tempo grafana/tempo \
  -n monitoring \
  -f k8s/observability/tempo/tempo-values.yaml
```

Create Pod Identity association:

```bash
aws eks create-pod-identity-association \
  --cluster-name $CLUSTER_NAME \
  --namespace monitoring \
  --service-account tempo \
  --role-arn arn:aws:iam::454143665149:role/TempoS3PodRole \
  --region $REGION || true
```

Restart Tempo:

```bash
kubectl delete pod tempo-0 -n monitoring --ignore-not-found
```

```bash
kubectl get pods -n monitoring | grep tempo
```

```bash
kubectl get svc tempo -n monitoring -o yaml | grep -E "3200|4317|4318"
```

---

## 23. Optional: Install telemetrygen Test Traces

Use only if real app tracing is not ready.

```bash
kubectl apply -f k8s/observability/tempo/telemetrygen-multi-services-cronjob.yaml
```

```bash
kubectl get cronjob -n monitoring | grep telemetrygen
```

Manual trigger:

```bash
kubectl create job telemetrygen-cart-manual \
  --from=cronjob/telemetrygen-cart-cron \
  -n monitoring || true
```

```bash
kubectl logs job/telemetrygen-cart-manual -n monitoring
```

---

# Grafana and Dashboards

## 24. Validate Dashboard JSON Files

```bash
ls -lh k8s/observability/grafana/dashboards/
```

```bash
wc -c k8s/observability/grafana/dashboards/*.json
```

```bash
jq '.title, .uid' k8s/observability/grafana/dashboards/*.json
```

If any file is `0` bytes or invalid, fix dashboards before continuing.

---

## 25. Apply Dashboard ConfigMap

```bash
kubectl apply -f k8s/observability/grafana/grafana-dashboards-configmap.yaml
```

```bash
kubectl get cm grafana-dashboards -n monitoring --show-labels
```

Expected label:

```text
grafana_dashboard=1
```

---

## 26. Install Grafana

```bash
helm upgrade --install grafana grafana/grafana \
  -n monitoring \
  -f k8s/observability/grafana/grafana-values.yaml
```

```bash
kubectl get pods -n monitoring | grep grafana
```

Check containers:

```bash
kubectl get pod -n monitoring -l app.kubernetes.io/name=grafana \
  -o jsonpath='{.items[0].spec.containers[*].name}{"\n"}'
```

Expected:

```text
grafana grafana-sc-dashboard
```

Restart Grafana:

```bash
kubectl delete pod -n monitoring -l app.kubernetes.io/name=grafana
```

```bash
kubectl get pods -n monitoring | grep grafana
```

---

## 27. Access Grafana

```bash
kubectl -n monitoring port-forward svc/grafana 3000:80
```

Open:

```text
http://localhost:3000
```

Login:

```text
Username: admin
Password: admin123
```

Go to:

```text
Dashboards -> Observability
```

Expected dashboards:

```text
Metrics Dashboard
Logs Dashboard
Traces Dashboard
```

---

# Optional Application Deployment

## 28. Deploy Retail UI

```bash
kubectl apply -f k8s/retail-ui-deployment.yaml
```

```bash
kubectl get pods -n application
```

```bash
kubectl get svc -n application
```

```bash
kubectl -n application port-forward svc/retail-ui 8080:80
```

Open:

```text
http://localhost:8080
```

---

# Validation

## 29. Validate Mimir

```bash
kubectl -n monitoring port-forward svc/mimir-gateway 9009:80
```

In another terminal:

```bash
curl -G http://localhost:9009/prometheus/api/v1/query \
  --data-urlencode 'query=up'
```

```bash
curl -G http://localhost:9009/prometheus/api/v1/query \
  --data-urlencode 'query=kube_pod_status_phase'
```

```bash
curl -G http://localhost:9009/prometheus/api/v1/query \
  --data-urlencode 'query=container_cpu_usage_seconds_total'
```

---

## 30. Validate Elasticsearch Logs

```bash
ES_PASS=$(kubectl -n logging get secret retail-logs-es-elastic-user -o jsonpath='{.data.elastic}' | base64 -d)
```

```bash
kubectl -n logging port-forward svc/retail-logs-es-http 9200:9200
```

In another terminal:

```bash
curl -k -u elastic:$ES_PASS https://localhost:9200/_cat/indices?v
```

```bash
curl -k -u elastic:$ES_PASS https://localhost:9200/_cat/indices?v | grep fluentbit
```

---

## 31. Validate Tempo

```bash
kubectl -n monitoring port-forward svc/tempo 3200:3200
```

In another terminal:

```bash
curl http://localhost:3200/ready
```

```bash
START=$(date -u -d '2 hours ago' +%s)
END=$(date -u +%s)
```

```bash
curl -G "http://localhost:3200/api/search" \
  --data-urlencode 'q={}' \
  --data-urlencode "start=$START" \
  --data-urlencode "end=$END" \
  --data-urlencode 'limit=20'
```

---

## 32. Final Status Check

```bash
kubectl get pods -n monitoring | egrep 'grafana|mimir|alloy|tempo|kube-state'
```

```bash
kubectl get pods -n logging | egrep 'fluent-bit|retail-logs|kibana'
```

```bash
kubectl get pods -n elastic-system
```

```bash
kubectl get pods -n application
```

```bash
kubectl get svc -n monitoring
```

```bash
kubectl get svc -n logging
```

```bash
kubectl get svc -n application
```

---

# Troubleshooting

## ImagePullBackOff / ErrImagePull

```bash
kubectl describe pod <POD_NAME> -n <NAMESPACE>
```

```bash
kubectl get pod <POD_NAME> -n <NAMESPACE> \
  -o jsonpath='{range .spec.initContainers[*]}{.name}{" -> "}{.image}{"\n"}{end}{range .spec.containers[*]}{.name}{" -> "}{.image}{"\n"}{end}'
```

```bash
aws ecr describe-images \
  --repository-name <REPOSITORY> \
  --image-ids imageTag=<TAG> \
  --region $REGION
```

Check S3 endpoint route tables:

```bash
aws ec2 describe-vpc-endpoints \
  --region $REGION \
  --vpc-endpoint-ids $S3_ENDPOINT_ID \
  --query 'VpcEndpoints[0].RouteTableIds' \
  --output table
```

---

## Grafana Dashboards Not Visible

Check sidecar:

```bash
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana \
  -c grafana-sc-dashboard \
  --tail=100
```

Check Grafana dashboard errors:

```bash
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana \
  -c grafana \
  --tail=200 | grep -i dashboard
```

If error is `EOF`, dashboard JSON files are empty or invalid:

```bash
wc -c k8s/observability/grafana/dashboards/*.json
jq '.title, .uid' k8s/observability/grafana/dashboards/*.json
```

---

## Mimir or Tempo Cannot Access S3

Check Pod Identity:

```bash
aws eks list-pod-identity-associations \
  --cluster-name $CLUSTER_NAME \
  --region $REGION
```

Check logs:

```bash
kubectl logs -n monitoring tempo-0 --tail=100
```

```bash
kubectl get pods -n monitoring | grep mimir
```

```bash
kubectl logs -n monitoring <MIMIR_POD_NAME> --tail=100
```

---

# Commit After Successful Restore

```bash
git status
```

```bash
git add .
```

```bash
git commit -m "Verify observability restore after cluster recreation"
```

```bash
git push
```
