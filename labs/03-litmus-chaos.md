# Lab 03: Litmus Chaos

In this lab we will see how can we do a more advanced experiment from templates located in the official Litmus "chaos hub".
The hypothesis is that if we have enough replicas of the application and our cluster is highly available there should be no downtime for the end users.

We will connect prometheus to litmus in order to see metrics directly on the litmus web interface.

# Deploy Litmus

1. Add the litmus helm repository
```
helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/
helm repo list
```

2. Create the namespace on which you want to install Litmus ChaosCenter
  - The chaoscenter components can be placed in any namespace, though it is typically placed in "litmus".
```
kubectl create ns litmus
```

3. Install Litmus ChaosCenter
```
helm install chaos litmuschaos/litmus --namespace=litmus
```

4. Verify installation
```
kubectl get pods -n litmus
```

5. By default, the service type is NodePort. For Ingress, we need to change the service type to ClusterIP for the `chaos-litmus-frontend-service` and `chaos-litmus-server-service` services
```
kubectl patch svc -n litmus chaos-litmus-frontend-service -p '{"spec": {"type": "ClusterIP"}}'
kubectl patch svc -n litmus chaos-litmus-server-service -p '{"spec": {"type": "ClusterIP"}}'
```

6. Get ingress controller LoadBalancer hostname
```
LB_HOSTNAME=$(kubectl -n ingress-nginx get svc ingress-nginx-controller -o json | jq -r .status.loadBalancer.ingress[0].hostname)
```

7. Define some variables for the route53 configuration file
```
DOMAIN=seladevops.com
HOSTED_ZONE_ID=Z217CC5WGOWD9G
```

8. Create route53 configuration file
```
tee -a ~/litmus-cname.json > /dev/null <<EOT
{
   "Comment":"CREATE CNAME record ",
   "Changes":[
      {
         "Action":"CREATE",
         "ResourceRecordSet":{
            "Name":"litmus.${DOMAIN}",
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

9. Create CNAME record in your Route53 Hosted Zone
```
aws route53 change-resource-record-sets --hosted-zone-id $HOSTED_ZONE_ID --change-batch file://~/litmus-cname.json
```

10. Set the environment variable INGRESS as true in the `chaos-litmus-server` deployment.
```
kubectl set env deployment chaos-litmus-server -n litmus --containers="graphql-server" INGRESS="true"
```

11. Create ingress manifest for litmus
```
tee -a ~/litmus-ingress.yaml> /dev/null <<EOT
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
  name: litmus-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: litmus.$DOMAIN
      http:
        paths:
          - backend:
              serviceName: chaos-litmus-frontend-service
              servicePort: 9091
            path: /(.*)
            pathType: ImplementationSpecific
          - backend:
              serviceName: chaos-litmus-server-service
              servicePort: 9002
            path: /backend/(.*)
            pathType: ImplementationSpecific
EOT
```

12. Apply litmus ingress
```
kubectl apply -f ~/litmus-ingress.yaml -n litmus
```
