# Runbook: EFK (Elasticsearch + Fluent Bit + Kibana) on AWS EKS using Helm + Kibana exposed via Service `LoadBalancer` (LAB)


>  EKS cluster →  EBS CSI storage →  ECK operator →  Elasticsearch + Kibana →  Fluent Bit shipping logs →  Sample app logs →  Kibana accessible from internet (LoadBalancer)

---

## 0) Prerequisites (run on your EC2 jump host / laptop)

### 0.1 Install required CLIs

Verify these commands work:

```bash
aws --version
kubectl version --client
eksctl version
helm version
```

### 0.2 AWS permissions required

Your AWS identity must be able to create:

* EKS cluster, nodegroup
* IAM roles/policies (for EBS CSI)
* CloudFormation stacks
* LoadBalancer (via Service type LoadBalancer)

### 0.3 Configure AWS region + credentials

```bash
aws configure
export AWS_REGION=ap-south-1
```

---

## 1) Create EKS cluster (your command)

```bash
eksctl create cluster --name my-lab-cluster --region ap-south-1 --node-type t2.medium
```

Wait until it finishes successfully.

### 1.1 Confirm kubectl is connected

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

---

## 2) Install EBS CSI Driver (this is mandatory for Elasticsearch PVC)

### 2.1 Confirm it’s NOT installed (expected initially)

```bash
kubectl get pods -n kube-system | grep ebs
```

Expected: no output

### 2.2 Associate IAM OIDC provider

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster my-lab-cluster \
  --region ap-south-1 \
  --approve
```

### 2.3 Create IAM service account for EBS CSI Controller (IRSA)

```bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster my-lab-cluster \
  --region ap-south-1 \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --role-name AmazonEKS_EBS_CSI_DriverRole
```

### 2.4 Install EBS CSI Driver as EKS Add-on

Get your AWS Account ID:

```bash
aws sts get-caller-identity --query Account --output text
```

Now install the addon (replace the Account ID if needed):

```bash
aws eks create-addon \
  --cluster-name my-lab-cluster \
  --addon-name aws-ebs-csi-driver \
  --region ap-south-1 \
  --service-account-role-arn arn:aws:iam::<ACCOUNT_ID>:role/AmazonEKS_EBS_CSI_DriverRole
```

### 2.5 Wait until addon is ACTIVE

```bash
aws eks describe-addon \
  --cluster-name my-lab-cluster \
  --addon-name aws-ebs-csi-driver \
  --region ap-south-1
```

Wait until:

```json
"status": "ACTIVE"
```

### 2.6 Verify CSI pods

```bash
kubectl get pods -n kube-system | grep ebs
```

Expected (names differ):

* `ebs-csi-controller... Running`
* `ebs-csi-node... Running`

---

## 3) Create gp3 StorageClass and make it default

> This ensures PVCs bind automatically.

Create file:

```bash
cat > gp3-sc.yaml <<'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  fsType: ext4
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
EOF
```

Apply:

```bash
kubectl apply -f gp3-sc.yaml
kubectl get storageclass
```

You should see:

* `gp3 (default)`

---

## 4) Install Elastic ECK Operator using Helm

### 4.1 Add Elastic Helm repo

```bash
helm repo add elastic https://helm.elastic.co
helm repo update
```

### 4.2 Install operator

```bash
helm install elastic-operator elastic/eck-operator \
  -n elastic-system --create-namespace
```

Verify:

```bash
kubectl get pods -n elastic-system
```

Expected:

* `elastic-operator-0 Running`

---

## 5) Install Elasticsearch + Kibana (ECK Stack) using Helm

### 5.1 Create values file (LAB-friendly resources + gp3 PVC)

```bash
cat > es-values.yaml <<'EOF'
elasticsearch:
  volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp3
        resources:
          requests:
            storage: 10Gi

  nodeSets:
    - name: default
      count: 1
      config:
        node.store.allow_mmap: false
      podTemplate:
        spec:
          containers:
            - name: elasticsearch
              resources:
                requests:
                  memory: 512Mi
                  cpu: "500m"
                limits:
                  memory: 1Gi
                  cpu: "1"

kibana:
  spec:
    podTemplate:
      spec:
        containers:
          - name: kibana
            resources:
              requests:
                memory: 256Mi
                cpu: "200m"
              limits:
                memory: 512Mi
                cpu: "500m"
EOF
```

### 5.2 Install ECK stack

```bash
helm install es-kb-quickstart elastic/eck-stack \
  -n elastic-stack --create-namespace \
  -f es-values.yaml
```

### 5.3 Verify PVC is Bound

```bash
kubectl get pvc -n elastic-stack -w
```

Expected:

* PVC goes `Pending → Bound`

### 5.4 Verify pods

```bash
kubectl get pods -n elastic-stack -w
```

Wait until:

* `elasticsearch-es-default-0 Running`
* `es-kb-quickstart-eck-kibana... Running`

### 5.5 Get Elasticsearch `elastic` password

```bash
kubectl get secret -n elastic-stack elasticsearch-es-elastic-user \
  -o go-template='{{.data.elastic | base64decode}}{{"\n"}}'
```

---

## 6) Install Fluent Bit using Helm (ships logs to Elasticsearch)

### 6.1 Create logging namespace

```bash
kubectl create namespace logging
```

### 6.2 Add Fluent Bit repo

```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update
```

### 6.3 Create Fluent Bit values

```bash
cat > fluent-bit-values.yaml <<'EOF'
serviceAccount:
  create: true
  name: fluent-bit

config:
  inputs: |
    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        multiline.parser  docker, cri
        Tag               kube.*
        Mem_Buf_Limit     10MB
        Skip_Long_Lines   On

  filters: |
    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Merge_Log           On
        Keep_Log            Off
        K8S-Logging.Parser  On
        K8S-Logging.Exclude On

  outputs: |
    [OUTPUT]
        Name            es
        Match           kube.*
        Host            elasticsearch-es-http.elastic-stack.svc
        Port            9200
        Logstash_Format On
        Logstash_Prefix kubernetes
        HTTP_User       elastic
        HTTP_Passwd     ${ES_PASSWORD}
        tls             On
        tls.verify      Off
        Retry_Limit     False
EOF
```

### 6.4 Export ES password into shell

```bash
export ES_PASSWORD=$(kubectl get secret -n elastic-stack elasticsearch-es-elastic-user \
  -o go-template='{{.data.elastic | base64decode}}')
```

### 6.5 Install Fluent Bit

```bash
helm install fluent-bit fluent/fluent-bit \
  -n logging \
  -f fluent-bit-values.yaml \
  --set-string env[0].name=ES_PASSWORD \
  --set-string env[0].value=$ES_PASSWORD
```

Verify:

```bash
kubectl get pods -n logging
```

Expected:

* Fluent Bit pods Running (1 per node)

---

## 7) Deploy sample app that generates logs

```bash
kubectl create namespace demo
```

Deploy:

```bash
kubectl apply -n demo -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: log-generator
spec:
  replicas: 2
  selector:
    matchLabels:
      app: log-generator
  template:
    metadata:
      labels:
        app: log-generator
    spec:
      containers:
      - name: app
        image: busybox
        command: ["/bin/sh","-c"]
        args:
          - |
            i=0;
            while true; do
              i=$((i+1));
              echo "$(date) INFO request_id=$i service=demo status=OK";
              if [ $((i % 5)) -eq 0 ]; then
                echo "$(date) ERROR request_id=$i service=demo status=FAILED";
              fi
              sleep 2;
            done
EOF
```

Verify app logs:

```bash
kubectl logs -n demo deploy/log-generator --tail=20
```

---

## 8) Expose Kibana to the Internet using Service type LoadBalancer (ECK way)

> Do **NOT** patch the Service directly — ECK will revert it.
> You must update the **Kibana CR**.

Edit Kibana CR:

```bash
kubectl edit kibana es-kb-quickstart-eck-kibana -n elastic-stack
```

Find this block:

```yaml
http:
  service:
    metadata: {}
    spec: {}
```

Change to:

```yaml
http:
  service:
    metadata: {}
    spec:
      type: LoadBalancer
```

Save and exit (`ESC`, then `:wq`).

### 8.1 Wait for external ELB DNS

```bash
kubectl get svc -n elastic-stack -w
```

You will see:

* Service type becomes `LoadBalancer`
* `EXTERNAL-IP` becomes an AWS ELB DNS name

Example:

```
xxxx.ap-south-1.elb.amazonaws.com
```

---

## 9) Access Kibana in Browser

Use **HTTPS + port 5601**:

```
https://<ELB-DNS>:5601
```

Expect a browser certificate warning (self-signed).
Proceed for LAB.

Login:

* Username: `elastic`
* Password: from Step 5.5

---

## 10) View Logs in Kibana (Discover)

### 10.1 Create Data View

In Kibana UI:

1. ☰ **Stack Management → Data Views**
2. **Create data view**
3. Set:

   * Name: `kubernetes-logs`
   * Index pattern: `kubernetes-*`
   * Time field: `@timestamp`
4. Save

### 10.2 Discover logs

Go to **Discover**

* Select data view: `kubernetes-logs`
* Time range: **Last 15 minutes**
* Filter:

  ```
  kubernetes.namespace_name : "demo"
  ```

You should see the INFO/ERROR logs from `log-generator`.

---

## 11) Troubleshooting (fast)

### A) Elasticsearch pod Pending with PVC issue

* Check CSI driver:

```bash
kubectl get pods -n kube-system | grep ebs
```

Must be Running.

* Check storageclass default:

```bash
kubectl get storageclass
```

Must show `gp3 (default)`.

### B) Fluent Bit running but logs not in Kibana

Check Fluent Bit logs:

```bash
kubectl logs -n logging -l app.kubernetes.io/name=fluent-bit --tail=100
```

### C) Kibana LoadBalancer opens but blank/empty response

Ensure you use:

```
https://<ELB-DNS>:5601
```

(not http, and don’t omit port)

---

## 12) Cleanup (LAB end)

Delete cluster:

```bash
eksctl delete cluster --name my-lab-cluster --region ap-south-1
```


---

**Runbook complete.**
