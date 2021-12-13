# Lab 02: Kubeinvaders Chaos

In this lab we will see how can we do a basic experiment which consists on deleting the pods of the demo application.
The hypothesis is that if we have enough replicas of the application there should be no downtime for the end users.

We will deploy also a visualization tool so that we can visualize the experiment.


## Deploy demo application

1. Create mongo and pacman namespaces
```
kubectl create ns mongo
kubectl create ns pacman
```

2. Deploy mongodb
```
kubectl apply -f sdp-chaos/demo-app/mongo -n mongo
```

3. Deploy pacman
```
kubectl apply -f sdp-chaos/demo-app/pacman -n pacman
```

4. Define some variables for the ingress and CNAME configuration
```
DOMAIN=<YOUR-DOMAIN>
HOSTED_ZONE_ID=<YOUR-HOSTED-ZONE-ID>
```

5. Get ingress controller LoadBalancer hostname
```
LB_HOSTNAME=$(kubectl -n ingress-nginx get svc ingress-nginx-controller -o json | jq -r .status.loadBalancer.ingress[0].hostname)
```

6. Create route53 configuration file
```
tee -a ~/pacman-cname.json > /dev/null <<EOT
{
   "Comment":"CREATE CNAME record ",
   "Changes":[
      {
         "Action":"CREATE",
         "ResourceRecordSet":{
            "Name":"pacman.${DOMAIN}",
            "Type":"CNAME",
            "TTL":300,
            "ResourceRecords":[
               {
                  "Value":"${LB_HOSTNAME}"
               }
            ]
         }
      }
   ]
}
EOT
```

7. Create CNAME record in your Route53 Hosted Zone
```
aws route53 change-resource-record-sets --hosted-zone-id $HOSTED_ZONE_ID --change-batch file://~/pacman-cname.json
```

8. Create ingress for the pacman demo app
```
cat <<EOT | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: pacman-ingress
  namespace: pacman
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    cert-manager.io/cluster-issuer: letsencrypt
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - pacman.$DOMAIN
      secretName: pacman-tls-secret
  rules:
  - host: pacman.$DOMAIN
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: pacman
            port:
              number: 80
EOT
```


## Deploy Kubeinvaders

1. Add the kubeinvaders repository to your helm repos
```
helm repo add kubeinvaders https://lucky-sideburn.github.io/helm-charts/
```

2. Define in what namespaces kubeinvaders can do chaos
```
NAMESPACES="pacman"
```

3. Create a namespace for kubeinvaders
```
kubectl create namespace kubeinvaders
```

4. Get ingress controller LoadBalancer hostname
```
LB_HOSTNAME=$(kubectl -n ingress-nginx get svc ingress-nginx-controller -o json | jq -r .status.loadBalancer.ingress[0].hostname)
```

6. Define some variables for the route53 configuration file
```
DOMAIN=<YOUR-DOMAIN>
HOSTED_ZONE_ID=<YOUR-HOSTED-ZONE-ID>
```

5. Create route53 configuration file
```
tee -a ~/cname.json > /dev/null <<EOT
{
   "Comment":"CREATE CNAME record ",
   "Changes":[
      {
         "Action":"CREATE",
         "ResourceRecordSet":{
            "Name":"kubeinvaders.${DOMAIN}",
            "Type":"CNAME",
            "TTL":300,
            "ResourceRecords":[
               {
                  "Value":"${LB_HOSTNAME}"
               }
            ]
         }
      }
   ]
}
EOT
```

6. Create CNAME record in your Route53 Hosted Zone
```
aws route53 change-resource-record-sets --hosted-zone-id $HOSTED_ZONE_ID --change-batch file://~/cname.json
```

5. Deploy kubeinvaders
```
helm upgrade --install kubeinvaders --set-string target_namespace=$NAMESPACES \
-n kubeinvaders kubeinvaders/kubeinvaders --set ingress.hostName=kubeinvaders.$DOMAIN
```

6. (Optional) With the default installation of nginx ingress controller adding ingressClassName is neded so you may have to edit the kubeinvaders ingress and add the following under spec:
```
ingressClassName: nginx
```

7. (Optional) To get an SSL certificate using cert-manager include the annotation `cert-manager.io/cluster-issuer: letsencrypt` and to enforce https `nginx.ingress.kubernetes.io/force-ssl-redirect: "true"` and include a tls section in the ingress configuration (cert-manager will create the secret).
  - For reference refer to the ingress configuration of pacman

## Deploy kube-ops-view

1. clone the kube-ops-view repo and cd into it
```
git clone https://codeberg.org/hjacobs/kube-ops-view.git
cd kube-ops-view
```

2. Deploy kube-ops-view
```
kubectl create ns kube-ops-view
kubectl apply -k deploy -n kube-ops-view
```

4. Get ingress controller LoadBalancer hostname
```
LB_HOSTNAME=$(kubectl -n ingress-nginx get svc ingress-nginx-controller -o json | jq -r .status.loadBalancer.ingress[0].hostname)
```

6. Define some variables for the route53 configuration file
```
DOMAIN=<YOUR-DOMAIN>
HOSTED_ZONE_ID=<YOUR-HOSTED-ZONE-ID>
```

5. Create route53 configuration file
```
tee -a ~/kubeops-cname.json > /dev/null <<EOT
{
   "Comment":"CREATE CNAME record ",
   "Changes":[
      {
         "Action":"CREATE",
         "ResourceRecordSet":{
            "Name":"kubeops.${DOMAIN}",
            "Type":"CNAME",
            "TTL":300,
            "ResourceRecords":[
               {
                  "Value":"${LB_HOSTNAME}"
               }
            ]
         }
      }
   ]
}
EOT
```

6. Create CNAME record in your Route53 Hosted Zone
```
aws route53 change-resource-record-sets --hosted-zone-id $HOSTED_ZONE_ID --change-batch file://~/kubeops-cname.json
```

8. Create ingress for kubeops-view
```
cat <<EOT | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kubeops-view-ingress
  namespace: kube-ops-view
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    cert-manager.io/cluster-issuer: letsencrypt
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - kubeops.$DOMAIN
      secretName: kubeops-tls-secret
  rules:
  - host: kubeops.$DOMAIN
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kube-ops-view
            port:
              number: 80
EOT
```

# Run Chaos Experiment

1. Open three browser windows and browse to:
  - kubeinvaders.$DOMAIN
  - kubeops.$DOMAIN
  - pacman.$DOMAIN

2. In the browser running kubeinvaders you can start the experiment automatically by clicking on the start button
  ![kubeinvaders](/images/kubeinvaders.png)

3. In the browser running kubeops you can write `namespace=pacman` to mark in green all the pods from the pacman namespace
  ![kubeops](/images/kubeops.png)

4. In the browser running pacman, start playing the game and you will see that even tho we are deleting the pods, the end users playing the game do not experience any downtime
  ![pacman](/images/pacman.png)