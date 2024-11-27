# TP-3 - Monistoring and Incidents Management #

* Monitor the application / platform

 - Install and configure Prometheus / Grafana / AlertManager stack
 - Install node_exporter and view metrics on Grafana
 - Install Loki/promtail (log centralization) - Ansible

The main goal of this workshop is to deploy Prometheus and Grafana into your Minikube cluster using Helm charts. 
Prometheus is used to monitor Kubernetes Cluster and other resources running on it. Globaly it is a systems and service monitoring tool.
Grafana helps to visualize metrics recorded by Prometheus and display them in dashboards.

All installation is done in 'monitoring' namespace. Ensure that minikube is started.

## Install prometheus using Official Helm Chart ##

Step 1 - Add prometheus repository : `helm repo add prometheus-community https://prometheus-community.github.io/helm-charts`

Step 2 - Install provided Helm chart for Prometheus : `helm install prometheus prometheus-community/prometheus`

Step 3 - Expose the prometheus-server service via NodePort : `kubectl expose service prometheus-server --type=NodePort --target-port=9090 --name=prometheus-service`

Step 4 - Access Prometheus UI : `minikube service prometheus-service --url -n monitoring`

## Repeat steps 1 - 4 for the Grafana component using Official Helm Chart ##

* Grafana Helm repo :  https://grafana.github.io/helm-charts

* Chart name : 'grafana/grafana'. Remember to set your own password for admin access.

* Expose Grafana service via NodePort in order to access Grafana UI with target port 3000.

* Access Grafana Web UI and configure a datasource with the deployed prometheus service url. 

* Install and Explore Node_Exporter Dashborad. ID 1860

## Configure AlertManager component ##

* Expose AlertManager service via NodePort in order to access UI with target port 9093.

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

```console
helm upgrade --install loki grafana/loki-stack --set promtail.enabled=false  --set grafana.enabled=false
```

Expose Loki service via NodePort in order to access UI with target port 3100 : `kubectl expose service loki --type=NodePort --target-port=3100 --name=loki-service`

Run `$LOKI_SERVICE_URL/ready` and check the response. 

## Install  Grafana/Loki Agent (promtail) by using Ansible ##

Configure Loki datasource on Grafana UI and run queries.
