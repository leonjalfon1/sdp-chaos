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
DOMAIN=<YOUR-DOMAIN>
HOSTED_ZONE_ID=<YOUR-HOSTED-ZONE-ID>
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

11. Deploy ingress for litmus
```
cat <<EOT | kubectl apply -f -
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /\$1
    cert-manager.io/cluster-issuer: letsencrypt
  name: litmus-ingress
  namespace: litmus
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - litmus.$DOMAIN
      secretName: litmuspreview-tls-secret
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

12. Deploy podMonitor for collecting chaos-exporter metrics
```
cat <<EOT | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: chaos-exporter-monitor
  namespace: monitoring
  labels:
    release: monitoring
spec:
  selector:
    matchLabels:
      app: chaos-exporter
  namespaceSelector:
    matchNames:
      - litmus
  podMetricsEndpoints:
    - port: tcp
    - interval: 1s
      metricRelabelings:
        - targetLabel: instance
          replacement: 'chaos-exporter-service'
EOT
```

## Setup Litmus

1. Open your browser and browse to: https://litmus.$DOMAIN
  ![initial-login](/images/litmus-setup.png)

2. Login using the following credentials:
  - Username: admin
  - Password: litmus

3. Create your own password to finish setup
  ![reset-password](/images/setup-password.png)

4. After the initial setup a `Chaos Agent` will be deployed into the cluster. This agent is what will perform the experiments in your cluster.
  - You can check the status of your `Chaos Agents` in the Litmus UI
  ![chaos-agent](/images/chaos-agent.png)

## Configure Prometheus as a Data Source for gathering Metrics

1. Browse to https://litmus.$DOMAIN, login and go to Observability > Data Sources
  ![data-source-1](/images/data-source-1.png)

2. Click on `Add Data Source` and configure as follows:
  ![data-source-2](/images/data-source-2.png)

3. In the next form leave all the defaults and click on `Save Changes`
  ![data-source-3](/images/data-source-3.png)

4. Once the `Data Source` is configured browse to Observability > Overview and click on `Create Dashboard`
  ![data-source-4](/images/data-source-4.png)

5. Select the `Node metrics` Dashboard
  ![data-source-5](/images/data-source-5.png)

6. Configure as follows
  ![data-source-6](/images/data-source-6.png)

7. In the next form, select all metrics
  ![data-source-7](/images/data-source-7.png)

8. In the next screen there is a problem with the promql query. to fix it change from `instance:node_cpu_utilisation:rate1m*100` to `instance:node_cpu_utilisation:rate5m*100` and click on `Save Changes`
  ![data-source-8](/images/data-source-8.png)

## Configure the Experiment

- In this experiment we are going to use some of the Built-in experiments of the `Litmus Chaos Hub`:
  - `Pod CPU Hog`: Consumes CPU resources on the application container
  - `Pod Memory Hog`: Consumes Memory resources on the application container
  - `Pod Delete`: Pod delete contains chaos to disrupt state of kubernetes resources.
  - `Pod Network Corruption`: Injects Network Packet Corruption into Application Pod

- The hypothesis is that the Application will be able to whitstand all the above conditions without any impact to the end users.

- To define the `Steady State` of the application we know that if a user browses to `https://pacman.$DOMAIN` the application should return Http status code 200

1. Browse to litmus.$DOMAIN, go to `Litmus Workflows` and click on the `Schedule a Worflow` button.
  ![experiment](/images/experiment-1.png)

2. Select the `Self-Agent` and click on `Next`
  ![experiment](/images/experiment-2.png)

3. Select the option `Import a workflow using YAML` and upload the chaos experiment YAML configuration located in this repository at `sdp-chaos/litmus/sdp-chaos.yaml` and click on `Next`
  ![experiment](/images/experiment-6.png)
  - For more info on how to configure experiments from `Litmus ChaosHub` refer to the section below.

4. Give a name and a description to your experiment and click on `Next`
  ![experiment](/images/experiment-3.png)

5. Select the `Schedule Now` option and click on `Next`
  ![experiment](/images/experiment-4.png)

6. Verify the experiment and click on `Finish`
  ![experiment](/images/experiment-5.png)

## Using Litmus ChaosHub experiments

One of the features of Litmus Chaos is that it has an official `ChaosHub` with experiments that are ready to use with minimal configuration.
In this Section we will configure a workflow with an experiment from the `Litmus ChaosHub`

1. Browse to litmus.$DOMAIN, go to `Litmus Workflows` and click on the `Schedule a Worflow` button.
  ![experiment](/images/experiment-1.png)

2. Select the `Self-Agent` and click on `Next`
  ![experiment](/images/experiment-2.png)

3. Select the option `Create a new workflow using the experiments from ChaosHubs`, choose `Litmus ChaosHub` from the drop down list and click `Next`
  ![chaos-hub](/images/chaos-hub-1.png)

4. Give a name and a description to your experiment and click on `Next`
  ![chaos-hub](/images/chaos-hub-2.png)

5. Click on the `Add a new experiment` button and choose your experiment. In this demo we will use `generic/pod-delete`.
  ![chaos-hub](/images/chaos-hub-3.png)

6. After the experiment has been added click on the edit button to configure it
  ![chaos-hub](/images/chaos-hub-4.png)

7. In general you can leave the defaults and click `Next`
  ![chaos-hub](/images/chaos-hub-11.png)

8. Configure the target application with the drop down lists and click `Next`
  ![chaos-hub](/images/chaos-hub-5.png)

9. In order to see if an experiment is succesful or not we need to let Litmus know what is the `Steady State`. To do this we configure `probes`. For this experiment we will use a continuous probe which checks that issuing a request to the application returns HTTP code 200. Click on `Add a new probe` and configure like below:
  ![chaos-hub](/images/chaos-hub-6.png)

10. Tune the experiment according to your needs. In this case we want to delete 60 percent of the Pods
  ![chaos-hub](/images/chaos-hub-7.png)

11. You can add more experiments by clicking again on `Add a new experiment`. When you are done click on `Next` and adjust the weight of each experiment. The final score of the experiment depends on these values.
  ![chaos-hub](/images/chaos-hub-8.png)

12. Select the `Schedule Now` option and click on `Next`.
  ![chaos-hub](/images/chaos-hub-9.png)

13. Verify the experiment and click on `Finish`
  ![chaos-hub](/images/chaos-hub-10.png)

- For more experiments and docs on how to use each one of them visit https://hub.litmuschaos.io/