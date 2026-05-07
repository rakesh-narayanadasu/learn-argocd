# ArgoCD Metrics Monitoring

In this we explore how to effectively visualize ArgoCD metrics using Prometheus and Grafana. ArgoCD exposes a comprehensive set of Prometheus metrics that can be scraped and visualized via a Grafana dashboard, offering deep insights into its performance and operational states.

## Configuring Prometheus to Scrape ArgoCD Metrics

ArgoCD provides various endpoints that expose Prometheus metrics for its critical components. To capture these metrics, Prometheus must be configured appropriately. The Prometheus Operator makes this process simpler by managing Prometheus, Alertmanager, Grafana, and other monitoring elements using Kubernetes custom resources.

### Viewing the Prometheus Configuration

To inspect the Prometheus configuration, execute commands inside the config-reloader container of the Prometheus pod. This allows you to verify global settings, scrape intervals, and the configured targets.

```bash theme={null}
$ kubectl exec -it prometheus-0 -c config-reloader -- /bin/sh
$ cat /etc/prometheus/config_out/prometheus.env.yaml
global:
  scrape_interval: 30s
rule_files:
  - /etc/prometheus/rules/prometheus-0/*.yaml
scrape_configs:
  - job_name: serviceMonitor/monitoring/kube-apiserver/0
```

> By default, Prometheus is set to scrape metrics from various Kubernetes components such as the kube-apiserver.

### Configuring Prometheus to Scrape ArgoCD Metrics

The Prometheus Operator leverages ServiceMonitor and PodMonitor custom resources to automatically discover and configure scraping targets. To enable service monitoring in ArgoCD, follow these steps:

1. **Expose Metrics in Services**\
   Ensure that the services exposing metrics are configured with an endpoint, port, and proper labels. ArgoCD includes services for the RepoServer, ArgoCD server, ServiceServer, and ApplicationSetController, each exposing its metrics.

2. **Create a ServiceMonitor**\
   A ServiceMonitor is a custom resource that instructs Prometheus how to discover metric-exposing services using matching labels. ArgoCD supplies a sample ServiceMonitor manifest ready for application in your Kubernetes cluster.

3. **Automatic Configuration Generation**\
   The Prometheus Operator uses the Prometheus custom resource to identify ServiceMonitors by their labels, subsequently generating the configuration required to scrape the defined services.

4. **Automatic Reload of Configuration**\
   Once ServiceMonitors are detected and processed, the Prometheus Operator triggers the ConfigReloader component. This action updates the Prometheus configuration to include new targets defined by the ServiceMonitors.

### Example Service and ServiceMonitor Configurations

Below you'll find an example of a service that exposes ArgoCD server metrics:

```yaml theme={null}
$ kubectl get svc argocd-server-metrics -o yaml
apiVersion: v1
kind: Service
metadata:
  name: argocd-server-metrics
  namespace: argocd
spec:
  ports:
    - name: metrics
      port: 8083
      protocol: TCP
      targetPort: 8083
  selector:
    app.kubernetes.io/name: argocd-server
```

Next, review the corresponding ServiceMonitor configuration that allows Prometheus to automatically discover the above service:

```yaml theme={null}
$ kubectl get servicemonitor argocd-server-metrics -o yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-server-metrics
  labels:
    release: prometheus-operator
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-server-metrics
  endpoints:
    - port: metrics
```

After applying these configurations, verify the updated Prometheus configuration. The ConfigReloader should now include additional scrape configurations for ArgoCD metrics:

```bash theme={null}
$ kubectl exec -it prometheus-0 -c config-reloader -- /bin/sh
$ cat /etc/prometheus/config_out/prometheus.env.yaml
global:
  scrape_interval: 30s
rule_files:
  - /etc/prometheus/rules/prometheus-0/*.yaml
scrape_configs:
  - job_name: serviceMonitor/monitoring/kube-apiserver/0
  - job_name: serviceMonitor/argocd/argocd-server-metrics/0
  - job_name: serviceMonitor/argocd/argocd-repo-server-metrics/0
  - job_name: serviceMonitor/argocd/argocd-applicationset-controller-metrics/0
```

> Ensure that all necessary ServiceMonitors and corresponding labels are correctly defined in your configurations to enable comprehensive metric scraping.

## Visualizing Metrics with Grafana

Once Prometheus begins scraping ArgoCD metrics, you can leverage Grafana to create insightful dashboards. Grafana connects to Prometheus as a data source and enables the creation of detailed, customizable dashboards that display:

* Operational metrics of various ArgoCD components
* Performance statistics and trends
* Alerts and notifications based on defined thresholds

You can use pre-built dashboards or develop custom ones tailored to your monitoring requirements.

***

# ArgoCD Metrics Alerts Monitoring

In this we explore how Prometheus can leverage ArgoCD metrics to raise alerts using Alertmanager automating the detection of synchronization issues in ArgoCD applications.

Alertmanager handles alerts generated by client applications, such as the Prometheus server. In this article, we explain how to configure alerting rules based on ArgoCD metrics within a Prometheus instance. Previously, we examined how Prometheus scrapes these metrics; now, we will use them to define alert rules for detecting synchronization problems in ArgoCD applications.

As part of the Prometheus Operator, configuration details are maintained for both Alertmanager and Prometheus rules. The Prometheus Rules custom resource enables you to define both recording and alerting rules for your Prometheus instance.

Below is an example of a PrometheusRule configuration containing a single alerting rule:

```yaml theme={null}
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  creationTimestamp: null
  labels:
    prometheus: example
    role: alert-rules
  name: prometheus-argocd-rules
spec:
- name: ArgoCD Rules
  rules:
  - alert: ArgoApplicationOutOfSync
    expr: argocd_app_info{sync_status="OutOfSync"} == 1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "{{ $labels.name }} Application has synchronization issue"
```

To verify the applied configuration, run the following commands. These commands connect to the Prometheus pod and display the contents of the rules configuration file:

```bash theme={null}
$ kubectl exec -it prometheus-0 -c config-reloader -- /bin/sh
$ cat /etc/prometheus/rules/prometheus-rulefiles-0/argocd-app-sync
```

Each alert rule configuration includes:

* **Alert name:** The identifier for the alert (e.g., "ArgoApplicationOutOfSync").
* **Expression:** The condition that triggers the alert. In this example, the alert fires when an ArgoCD application's synchronization status is out of sync (`argocd_app_info{sync_status="OutOfSync"} == 1`).
* **Duration:** The period (5 minutes in this instance) the condition must persist before the alert triggers.
* **Labels:** Metadata tags for routing and categorizing alerts, such as a `severity` level set to "warning".
* **Annotations:** Additional information, including a summary message to describe the alert.


When the Prometheus rule is created, the Prometheus Operator automatically generates the corresponding configuration for Prometheus using the Prometheus Rule custom resource definition (CRD). It then uses the Config Reloader component to update the rules file accordingly.

Below is an updated version of the PrometheusRule configuration (functionally identical to the previous example but formatted properly):
```bash
vi argocd-rules.yaml
```

```yaml theme={null}
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: prometheus-argocd-rules
  namespace: monitoring
  labels:
    release: my-kube-prometheus-stack   # IMPORTANT
spec:
  groups:
  - name: argocd.rules
    rules:
    - alert: ArgoApplicationOutOfSync
      expr: argocd_app_info{sync_status="OutOfSync"} == 1
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "'{{ $labels.name }}' Application has synchronization issue"
```

You can inspect the active configuration with these commands:

```bash theme={null}
# verify
kubectl get prometheusrule -n monitoring | grep argocd

kubectl exec -it prometheus-my-kube-prometheus-stack-prometheus-0 -n monitoring -- sh

ls /etc/prometheus/rules/prometheus-my-kube-prometheus-stack-prometheus-rulefiles-0/

grep -i argocd /etc/prometheus/rules/prometheus-my-kube-prometheus-stack-prometheus-rulefiles-0/*
```

> After updating the rules file, check the Rules tab in the Prometheus UI. If an ArgoCD application's synchronization status deviates from the expected state, the alert will trigger and be visible in the Alertmanager UI.

***

# Monitoring through Prometheus Grafana

In this you will learn how to utilize Prometheus and Grafana to visualize ArgoCD metrics. ArgoCD exposes a variety of Prometheus metrics across its different components including the application controller, API server, repository server, and applicationset controller. This guide walks you through inspecting these metrics within your cluster, setting up Prometheus and Grafana using Helm, configuring ServiceMonitor resources to scrape ArgoCD metrics, and visualizing the collected data.

## Examining ArgoCD Metrics

ArgoCD’s documentation provides detailed descriptions of the metrics exposed by its components. For instance, the following image illustrates the metrics for the ArgoCD application controller, including names, types, and descriptions:

Similarly, the API server provides its own set of metric endpoints:

To inspect all ArgoCD services in your cluster, run the following command to list the services in the ArgoCD namespace:

```bash theme={null}
kubectl -n argocd get svc
```

You might see output similar to this:

```plaintext theme={null}
NAME                                     TYPE        CLUSTER-IP      EXTERNAL-IP  PORT(S)                     AGE
argocd-applicationset-controller         ClusterIP   10.100.58.34    <none>       7000/TCP,8080/TCP           29h
argocd-dex-server                        ClusterIP   10.109.179.192  <none>       5556/TCP,5557/TCP,5558/TCP   29h
argocd-metrics                           ClusterIP   10.100.111.162  <none>       8082/TCP                    29h
argocd-notifications-controller-metrics   ClusterIP   10.110.116.143  <none>       9001/TCP                    29h
argocd-redis                             ClusterIP   10.106.239.172  <none>       6379/TCP                    29h
argocd-repo-server                       ClusterIP   10.101.4.27     <none>       8081/TCP,8084/TCP           29h
argocd-server                            NodePort    10.98.110.228   <none>       80:30663/TCP,443:31194/TCP   29h
argocd-server-metrics                    ClusterIP   10.97.180.219   <none>       8083/TCP                    29h
```

Accessing the server’s metrics using a curl command on the default port (80) may return a “404 page not found” error:

```bash theme={null}
curl 10.98.110.228:80/metrics
```

Instead, use the dedicated metrics endpoint on the `argocd-metrics` service (port 8082) to retrieve Prometheus metrics. You might see output similar to:

```plaintext theme={null}
workqueue_work_duration_seconds_bucket{name="app_reconciliation_queue",le="1e-05"} 413
workqueue_work_duration_seconds_bucket{name="app_reconciliation_queue",le="0.001"} 5333
workqueue_work_duration_seconds_bucket{name="app_reconciliation_queue",le="0.01"} 5978
...
workqueue_work_duration_seconds_count{name="project_reconciliation_queue"} 1225
```

## Setting Up Prometheus and Grafana

To visualize ArgoCD metrics, you must deploy a Prometheus server. The Kube Prometheus Stack available on Artifact Hub provides Prometheus, Grafana, Alertmanager, and other essential custom resource definitions.

### Adding the Prometheus Helm Repository

First, add the Prometheus community repository and update your local Helm repository cache:

```bash theme={null}
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### Installing the Prometheus Stack

Create a new namespace for monitoring (e.g., `monitoring`) and install the Prometheus stack. Depending on your requirements, you can choose between different chart versions. Here are two examples:

Using a recent version (e.g., 40.1.2):

```bash theme={null}
kubectl create ns monitoring
helm install my-kube-prometheus-stack prometheus-community/kube-prometheus-stack --version 40.1.2 -n monitoring
```

Or using an alternative version (e.g., 35.3.0):

```bash theme={null}
kubectl create ns monitoring
helm install my-kube-prometheus-stack prometheus-community/kube-prometheus-stack --version 35.3.0 -n monitoring
```

The installation process creates multiple resources (services, pods, daemonsets, statefulsets, etc.). To view the Prometheus service (set as NodePort), run:

```bash theme={null}
kubectl -n monitoring get svc
```

You can then access the Prometheus UI by navigating to the node’s IP with the assigned NodePort (e.g., 31534). In the Prometheus interface, go to “Status” → “Targets” to see the list of scraped endpoints.

```bash
kubectl port-forward svc/my-kube-prometheus-stack-prometheus 9090:9090 -n monitoring
```

Filtering the target list for ArgoCD-related endpoints should display the newly added targets

## Configuring ServiceMonitors for ArgoCD

To allow Prometheus to scrape ArgoCD metrics, create ServiceMonitor resources in the ArgoCD namespace. These YAML manifests specify the services to be monitored and must include a label that matches the Prometheus operator’s selector (in this case, the release name “my-kube-prometheus-stack”).

Below is an example configuration with three ServiceMonitors—for general metrics, server metrics, and repository server metrics. Similar manifests can be created for monitoring the applicationset controller.

```yaml theme={null}
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-metrics
  labels:
    release: my-kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-metrics
  endpoints:
    - port: metrics
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-server-metrics
  labels:
    release: my-kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-server-metrics
  endpoints:
    - port: metrics
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-repo-server-metrics
  labels:
    release: my-kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-repo-server
  endpoints:
    - port: metrics
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-applicationset-controller-metrics
  labels:
    release: my-kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-applicationset-controller
  endpoints:
    - port: metrics

```

Apply these manifests to the ArgoCD namespace (replace `argocd-service-monitors.yaml` with your filename):

```bash theme={null}
kubectl -n argocd apply -f argocd-service-monitors.yaml
```

Verify the ServiceMonitors are set up correctly:

```bash theme={null}
kubectl -n argocd get servicemonitors
```

You should see output similar to:

```plaintext theme={null}
NAME                                      AGE
argocd-applicationset-controller-metrics  11s
argocd-metrics                            11s
argocd-repo-server-metrics                11s
argocd-server-metrics                     11s
```

Ensure that the metadata label “release” in each ServiceMonitor exactly matches the Helm release name used during the Prometheus stack installation (e.g., “my-kube-prometheus-stack”). You can confirm the Prometheus operator’s selector labels by running:

```bash theme={null}
kubectl -n monitoring get prometheus.monitoring.coreos.com -o yaml | grep -i servicemonitorselector -A5
```

## Visualizing ArgoCD Metrics

After the ServiceMonitors are active, Prometheus automatically adds ArgoCD targets. In the Prometheus UI, search for “argocd” (for example, using a query like `argocd_app_info`) to display detailed metrics and statuses for your applications.

## Using Grafana for Visualization

Grafana offers rich visual dashboards to analyze these metrics. The recommended ArgoCD Grafana dashboard displays application health, sync status, component statistics, and more.

### Steps to Access the Grafana Dashboard

1. **Identify the Grafana Service:**\
   Locate the Grafana service in the `monitoring` namespace. By default, it is a ClusterIP service; modify it to a NodePort for external access if needed. Use the following command to edit the service:

   ```bash theme={null}
   kubectl -n monitoring edit svc my-kube-prometheus-stack-grafana
   ```

   ```bash
   kubectl port-forward svc/my-kube-prometheus-stack-grafana 8050:80 -n monitoring
   ```

2. **Access Grafana:**\
   After changing the service to NodePort (e.g., port 31762), navigate to the URL:\
   http\://\<node-ip>:31762

3. **Log In:**\
   The default username is `admin`. Retrieve the password from the Grafana secret using:

   ```bash theme={null}
   kubectl -n monitoring get secret my-kube-prometheus-stack-grafana -o json | jq -r '.data["admin-password"]' | base64 --decode
   ```

4. **Import the ArgoCD Dashboard:**\
   Click on the “+” icon in Grafana and select “Import.” You can import the ArgoCD dashboard using its ID or by supplying a JSON file. For example, import directly from this link:\
   [ArgoCD Grafana Dashboard](https://grafana.com/grafana/dashboards/14584-argocd/)

After importing, you will see a comprehensive dashboard that displays metrics such as application status, sync performance, and overall health.

This Grafana dashboard provides actionable insights for both operations and development teams by summarizing the health, performance, and synchronization status of your ArgoCD-managed applications.

***

# Raise Alert using AlertManager

In this guide, you will learn how to raise ArgoCD alerts and view them in Alertmanager. Previously, we deployed a Kubernetes monitoring stack that included Alertmanager. In this we will update Alertmanager to use a NodePort service and add a custom Prometheus alert based on ArgoCD metrics.

Follow this step-by-step procedure to configure and validate your alerts.

## Step 1: Update Alertmanager Service to NodePort

First, update the Alertmanager service so that it exposes a NodePort. In this example, the NodePort for Alertmanager is set to 32501.

To modify the Alertmanager service, run:

```bash theme={null}
kubectl -n monitoring edit svc my-kube-prometheus-stack-alertmanager
```

Next, verify the service details using:

```bash theme={null}
kubectl -n monitoring get svc
```

The expected output includes:

```text theme={null}
NAME                                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                     AGE
alertmanager-operated                     ClusterIP   None            <none>        9093/TCP,9094/TCP,9094/UDP 25m
my-kube-prometheus-stack-alertmanager     NodePort    10.109.208.61   <none>       9093:32501/TCP              25m
my-kube-prometheus-stack-grafana          ClusterIP   10.108.205.232  <none>       80:31762/TCP                25m
my-kube-prometheus-stack-kube-state-metrics ClusterIP  10.107.154.159  <none>       8080/TCP                    25m
my-kube-prometheus-stack-prometheus       ClusterIP   10.101.105.243  <none>       443/TCP                     25m
my-kube-prometheus-stack-prometheus-node-exporter NodePort 10.104.79.233  <none>       9090:31534/TCP              25m
prometheus-operated                        ClusterIP   10.103.115.32   None         9100/TCP                    25m
```

This confirms that the Alertmanager service is now accessible via NodePort 32501.

```bash
kubectl port-forward svc/my-kube-prometheus-stack-alertmanager 9093:9093 -n monitoring
```

## Step 2: Explore Existing Prometheus Alert Rules

Now, review the existing Prometheus alert rules to identify if there are any rules related to ArgoCD. Open the Prometheus UI and navigate to the rules tab. While you will see several rules like AlertmanagerFailedReload, AlertmanagerMembersInconsistent, and AlertmanagerFailedToSendAlerts, none are specifically configured for ArgoCD.

For instance, an existing set of rules appears as follows:

```yaml theme={null}
alert: AlertmanagerFailedReload
expr: max_over_time(alertmanager_config_last_reload_successful{job="my-kube-prometheus-stack-alertmanager", namespace="monitoring"}[5m]) == 0
for: 10m
labels:
  severity: critical
annotations:
  description: Configuration has failed to load for {{ $labels.namespace }}{{ $labels.pod }}.
  runbook_url: https://runbooks.prometheus-operator.dev/runbooks/alertmanager/alertmanagerfailedreload
  summary: Reloading an Alertmanager configuration has failed.

alert: AlertmanagerMembersInconsistent
expr: max_over_time(alertmanager_cluster_members{job="my-kube-prometheus-stack-alertmanager", namespace="monitoring"}[5m]) < on(namespace, service) group_left() count by(namespace, service) (max_over_time(alertmanager_cluster_members{job="my-kube-prometheus-stack-alertmanager", namespace="monitoring"}[5m]))
for: 15m
labels:
  severity: critical
annotations:
  description: Alertmanager {{ $labels.namespace }}{{ $labels.pod }} has only found {{ $value }} members of the {{ $labels.job }} cluster.
  runbook_url: https://runbooks.prometheus-operator.dev/runbooks/alertmanager/alertmanagermembersinconsistent
  summary: A member of an Alertmanager cluster has not found all other cluster members.

alert: AlertmanagerFailedToSendAlerts
expr: (rate(alertmanager_notifications_failed_total{job="my-kube-prometheus-stack-alertmanager", namespace="monitoring"}[5m]) > 0.01)
for: 5m
labels:
  severity: warning
```

Since there is no specific rule for ArgoCD, we will add a custom alert rule next.

## Step 3: Create a Custom ArgoCD Alert Rule

Add a custom alert that triggers when an ArgoCD application is out of sync. This alert uses the Prometheus metric `argocd_app_info` (assumed to be available) to check if the sync status is "OutOfSync." If this condition persists for one minute, the alert is raised with a warning severity.

To add the new alert rule, edit the Prometheus rules in the monitoring namespace:

```bash theme={null}
kubectl -n monitoring edit prometheusrule my-kube-prometheus-stack-alertmanager.rules
```

Append the following configuration at the end of the file:

```yaml theme={null}
- alert: AlertmanagerClusterCrashlooping
  annotations:
    description: '{{ $value | humanizePercentage }} of Alertmanager instances within the {{ $labels.job }} cluster have restarted at least 5 times in the last 10m.'
    runbook_url: https://runbooks.prometheus-operator.dev/runbooks/alertmanager/alertmanagerclustercrashlooping
  expr: |
    (
      count by (namespace, service) (
        changes(process_start_time_seconds{job="my-kube-prometheus-stack-alertmanager", namespace="monitoring"}[10m])
      )
    )
    /
    count by (namespace, service) (
      up{job="my-kube-prometheus-stack-alertmanager", namespace="monitoring"}
    ) > 0.5
  for: 5m
  labels:
    severity: critical

- alert: ArgoApplicationOutOfSync
  expr: argocd_app_info{sync_status="OutOfSync"} == 1
  for: 1m
  labels:
    severity: warning
  annotations:
    summary: "'{{ $labels.name }}' Application has synchronization issue"
```

When you save your changes, the Prometheus operator will update the configuration to include these new rules.

To verify the update, run:

```bash theme={null}
kubectl -n monitoring get prometheusrules my-kube-prometheus-stack-alertmanager.rules -o yaml | grep -i argocd
```

If needed, you can edit the `'prometheusrule'` again:

```bash theme={null}
kubectl -n monitoring edit prometheusrule my-kube-prometheus-stack-alertmanager.rules
```

## Step 4: Validate the New Alert Rule

After updating the rules, check the Prometheus UI to confirm that the new ArgoCD alert appears alongside other Alertmanager rules. The rule for `ArgoApplicationOutOfSync` should be visible similar to the examples below:

```yaml theme={null}
alert: AlertmanagerFailedReload
expr: max_over_time(alertmanager_config_last_reload_successful{job="my-kube-prometheus-stack-alertmanager", namespace="monitoring"}[5m]) == 0
for: 10m
labels:
  severity: critical
annotations:
  description: Configuration has failed to load for {{ $labels.namespace }}{{ $labels.pod }}.
  runbook_url: https://runbooks.prometheus-operator.dev/runbooks/alertmanager/alertmanagerfailedreload
  summary: Reloading an Alertmanager configuration has failed.
---
alert: AlertmanagerMembersInconsistent
expr: max_over_time(alertmanager_cluster_members{job="my-kube-prometheus-stack-alertmanager", namespace="monitoring"}[5m]) < on(namespace, service) group_left() count by(namespace, service) (max_over_time(alertmanager_cluster_members{job="my-kube-prometheus-stack-alertmanager", namespace="monitoring"}[5m]))
for: 15m
labels:
  severity: critical
annotations:
  description: Alertmanager {{ $labels.namespace }}{{ $labels.pod }} has only found {{ $value }} members of the {{ $labels.job }} cluster.
  runbook_url: https://runbooks.prometheus-operator.dev/runbooks/alertmanager/alertmanagermembersinconsistent
  summary: A member of an Alertmanager cluster has not found all other cluster members.
---
alert: AlertmanagerFailedToSendAlerts
expr: (rate(alertmanager_notifications_failed_total{job="my-kube-prometheus-stack-alertmanager", namespace="monitoring"}[5m]) > 0.01)
for: 5m
labels:
  severity: warning
annotations:
  summary: Alertmanager is failing to send alerts.
---
alert: ArgoApplicationOutOfSync
expr: argocd_app_info{sync_status="OutOfSync"} == 1
for: 1m
labels:
  severity: warning
annotations:
  summary: "'{{ $labels.name }}' Application has synchronization issue"
```

## Step 5: Test the Alert by Simulating an Out-of-Sync Application

To trigger the new alert, simulate an ArgoCD application going out of sync. For example, modify the replica count in a deployment definition. The following is an example for a deployment named "solar-system":

```yaml theme={null}
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: solar-system
  name: solar-system
spec:
  replicas: 0
  selector:
    matchLabels:
      app: solar-system
  template:
    metadata:
      labels:
        app: solar-system
    spec:
      containers:
      - image: siddharth67/solar-system:v9
        name: solar-system
        imagePullPolicy: Always
        ports:
        - containerPort: 80
```

After applying this change, the application status will change to "OutOfSync" and the `ArgoApplicationOutOfSync` alert should trigger within one minute.

Refresh the Alertmanager UI to view the generated alerts. You should see alerts for all out-of-sync ArgoCD applications. For example, the interface might display multiple alerts related to your ArgoCD applications.

In this you updated the Alertmanager service to use a NodePort, added a custom Prometheus alert for an ArgoCD application that went out of sync, and verified the new alert through the Alertmanager UI. This methodology can be extended to monitor and alert on additional ArgoCD scenarios.

