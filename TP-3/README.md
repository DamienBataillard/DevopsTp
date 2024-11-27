# TP-3 - Monistoring and Incidents Management #

* Monitor the application / platform

 - Install and configure Prometheus / Grafana / AlertManager stack
 - Install node_exporter and view metrics on Grafana
 - Install and configure Loki/promtail with Ansible

The main goal of this workshop is to deploy Prometheus and Grafana into your Minikube cluster using Helm charts. 
Prometheus is used to monitor Kubernetes Cluster and other resources running on it. Globaly it is a systems and service monitoring tool.
Grafana helps to visualize metrics recorded by Prometheus and display them in dashboards.

All installation is done in 'monitoring' namespace. Ensure that minikube is started.

## Install prometheus using Official Helm Chart ##

Step 1 - Add prometheus repository : `helm repo add prometheus-community https://prometheus-community.github.io/helm-charts`

Step 2 - Install provided Helm chart for Prometheus : `helm install prometheus prometheus-community/prometheus`

Step 3 - Get the Prometheus server URL by running these commands in the same shell
```shell
  export POD_NAME=$(kubectl get pods --namespace monitoring -l "app.kubernetes.io/name=prometheus,app.kubernetes.io/instance=prometheus" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace monitoring port-forward $POD_NAME 9090
```

Step 4 - Access Prometheus UI : `minikube service prometheus-service --url -n monitoring`

## Repeat steps 1 - 4 for the Grafana component using Official Helm Chart ##

* Grafana Helm repo :  https://grafana.github.io/helm-charts

* Chart name : 'grafana/grafana'. Remember to set your own password for admin access.

* Get the Grafana URL to visit by running these commands in the same

 ```shell
export POD_NAME=$(kubectl get pods --namespace monitoring -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}")
     kubectl --namespace monitoring port-forward $POD_NAME 3000
```

* Access Grafana Web UI and configure a datasource with the deployed prometheus service url. 

* Install and Explore Node_Exporter Dashborad. ID 1860

## Configure AlertManager component ##

* Get the Alertmanager URL by running these commands in the same shell :
  ```shell
  export POD_NAME=$(kubectl get pods --namespace monitoring -l "app.kubernetes.io/name=alertmanager,app.kubernetes.io/instance=prometheus" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace monitoring port-forward $POD_NAME 9093
  ```
* Analyze the content below, update the default prometheus configuration to set up an alert:

```yaml
serverFiles:
  alerting_rules.yml:
    groups:
      - name: Instances
        rules:
          - alert: InstanceDown
            expr: up == 0
            for: 1m
            labels:
              severity: critical
            annotations:
              description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minutes."
              summary: "Instance {{ $labels.instance }} down"
```

Apply the configuration update with : `helm upgrade --reuse-values -f prometheus-alerts-rules.yaml prometheus prometheus-community/prometheus`

Delete the 'prometheus-prometheus-pushgateway' deployment and see that you have an alert in AlertManager UI.

## Configure AlertManager to send Alerts by Email ##

Create the file *alertmanager-config.yaml* with the configuration and apply it with the following command ( Official email_config : https://prometheus.io/docs/alerting/latest/configuration/#email_config):

```console
   helm upgrade --reuse-values -f alertmanager-config.yaml prometheus prometheus-community/prometheus
```

## Configure Grafana/Loki ##

Loki is Grafana’s log aggregation component. Inspired by Prometheus, it’s highly scalable and capable of handling petabytes of log data.

Install the component by using the  Helm chart provided by grafana :
Create a file with the following contents:
```yaml
loki:
  commonConfig:
    replication_factor: 1
  schemaConfig:
    configs:
      - from: "2024-04-01"
        store: tsdb
        object_store: s3
        schema: v13
        index:
          prefix: loki_index_
          period: 24h
  pattern_ingester:
      enabled: true
  limits_config:
    allow_structured_metadata: true
    volume_enabled: true
    retention_period: 672h # 28 days retention
  compactor:
    retention_enabled: true 
    delete_request_store: s3
  ruler:
    enable_api: true

minio:
  enabled: true
      
deploymentMode: SingleBinary

singleBinary:
  replicas: 1

# Zero out replica counts of other deployment modes
backend:
  replicas: 0
read:
  replicas: 0
write:
  replicas: 0

ingester:
  replicas: 0
querier:
  replicas: 0
queryFrontend:
  replicas: 0
queryScheduler:
  replicas: 0
distributor:
  replicas: 0
compactor:
  replicas: 0
indexGateway:
  replicas: 0
bloomCompactor:
  replicas: 0
bloomGateway:
  replicas: 0
```

... then install with the command:

```console
helm install loki grafana/loki -f values.yaml -n monitoring
```

Expose Loki service  in order to access UI with target port 3100 : `kubectl port-forward --namespace monitoring svc/loki-gateway 3100:80`

To continue the exercise, follow the instructions displayed on the console after installation.

## Install  Grafana/Loki Agent (promtail) by using Ansible ##

Configure Loki datasource on Grafana UI and run queries.
