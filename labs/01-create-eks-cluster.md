## Lab 01: Create EKS Cluster

In this lab we will create the environment where all the tools and demo applications will be deployed.


# Create Cluster

1. Download eksctl
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv -v /tmp/eksctl /usr/local/bin
```
2. Enable completion (Optional)
```
eksctl completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
```

3. Define some variables for the cluster config
```
CLUSTER_NAME=chaos
AWS_REGION=eu-central-1

```

4. Create cluster config
```
tee -a ~/cluster-config.yaml > /dev/null <<EOT
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: $CLUSTER_NAME
  region: $AWS_REGION
  version: "1.20"
availabilityZones: ["${AWS_REGION}a", "${AWS_REGION}b", "${AWS_REGION}c"]
managedNodeGroups:
- name: nodegroup
  desiredCapacity: 3
  instanceType: t2.medium
  ssh:
    enableSsm: true
EOT
```

5. Create cluster
```
eksctl create cluster -f ~/cluster-config.yaml
```

6. Make sure your kubectl is configured with the new cluster as the current-context
```
kubectl config current-context
```

## Deploy Third Parties

### Nginx ingress controller

1. Deploy nginx ingress controller 
```
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

### Kube Prometheus Stack

1. Add the prometheus-community repo to your Helm repositories
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

2. Deploy the kube-prometheus-stack
```
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
```

### Cert Manager

1. Add the jetstack repository to your Helm repositories
```
helm repo add jetstack https://charts.jetstack.io
```

2. Create `cert-manager` namespace
```
kubectl create namespace cert-manager
```

3. Install cert-manager
```
helm install cert-manager jetstack/cert-manager --namespace cert-manager --set installCRDs=true
```

6. Configure a variable with your email
```
EMAIL=<YOUR-EMAIL>
```

5. Deploy a cluster issuer to get valid ssl certificates from Letsencrypt
```
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
   name: letsencrypt
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: $EMAIL
    privateKeySecretRef:
      name: letsencrypt
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```

