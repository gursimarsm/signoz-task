# Monitor Kubernetes Pods with Signoz

In this guide, we’ll configure SigNoz to send Slack alerts when any Kubernetes pod remains in the Pending state for more than 5 minutes.

## Step 1: Spin Up a Kubernetes Cluster & Install Helm

* Refer this [guide](https://www.armosec.io/blog/setting-up-kubernetes-cluster/) if you wish to learn how to spin up a K8s cluster.

## Step 2: Sign Up for SigNoz (It's Free!)

SigNoz is like a dashboard for your applications. It shows you what's happening and can send alerts.
SigNoz Cloud is where we’ll send all the telemetry data.

1.  Head to [SigNoz Cloud](https://signoz.io) and sign up.
3.  Once inside, head to New Data Source/New Source, then select your desired data source (Kubernetes Pod Logs in this case) and copy down:
    -   Your **Ingestion Key** (authenticates the cluster).
    -   Your **Region Endpoint** (Ex: `ingest.<REGION>.signoz.cloud:443`).
We’ll use these values to configure our Helm chart.

## Step 3: Install the SigNoz K8s-Infra Agent

To start collecting metrics and logs, we’ll deploy the **K8s-Infra Helm chart**.
This chart:
-   Installs the **OpenTelemetry Collector** as a DaemonSet.
-   Collects **host/node/pod metrics** (CPU, memory, disk).
-   Collects **pod logs** (enabled by default).
-   Optionally collects **Kubernetes events** (like scheduling failures).
-   Exports everything to **SigNoz Cloud** over OTLP.

**Note: We don’t need to install OTel separately.**

For more info about the agent, visit: [Link](https://signoz.io/docs/collection-agents/k8s/k8s-infra/overview/)

### Add the Helm repo

`helm repo add signoz https://charts.signoz.io`

`helm repo update` 

### Create an override file

We’ll create `override-values.yaml` to configure the cluster name, region endpoint, and logs/metrics settings.

```yaml
global:
  cloud: others
  clusterName: <CLUSTE_NAME>
  deploymentEnvironment: <DEPLOYMENT_ENVIRONMENT>

otelCollectorEndpoint: ingest.<REGION>.signoz.cloud:443
otelInsecure: false
signozApiKey: <SIGNOZ_INGESTION_KEY>

presets:
  otlpExporter:
    enabled: true

  # Metrics collection
  hostMetrics:
    enabled: true
  kubeletMetrics:
    enabled: true
  clusterMetrics:
    enabled: true
  kubernetesAttributes:
    enabled: true

  # Logs collection
  logsCollection:
    enabled: true
    whitelist:
      enabled: true
      namespaces:
        - <NAMESPACE>
      pods: []
      containers: []

  # Events
  k8sEvents:
    enabled: true
```

Note
-   Replace  `<SIGNOZ_INGESTION_KEY>`  with the one provided by SigNoz.
-   Replace  `<CLUSTER_NAME>`  with the name of the Kubernetes cluster or a unique identifier of the cluster.
-   Replace  `<DEPLOYMENT_ENVIRONMENT>`  with the deployment environment of your application. Example: "staging", "production", etc.
- Replace `<NAMESPACE>`  with the namespace of your application. By default, logs are collected from almost all namespaces.

### Install the Helm chart

`helm install my-release signoz/k8s-infra -n signoz --create-namespace -f override-values.yaml`

#### Check the pods:

`kubectl get pods -n signoz` 

## Step 4: Verify Metrics & Logs in SigNoz Cloud

1.  **Wait 2-3 minutes**  for data to start flowing
    
2.  Log in to your SigNoz dashboard.
    
3.  **Go to "Metrics"**  in the left menu.
    
4.  Search for **"k8s.pod"** , you should see metrics appearing.


## Step 5: Import a Kubernetes Dashboard

Instead of building from scratch, SigNoz provides templates.

-  Navigate to Dashboards → + New Dashboard → View Templates
-  Select Kubernetes → Kubernetes Pod Metrics – Detailed → Download JSON (or) Copy JSON
-  Navigate back to Dashboards  → + New Dashboard  → Import JSON  → Paste / Upload  → Import & Next

## Step 6: Set Up Slack Notifications

Now we'll connect Slack so you get instant notifications when pods are stuck.

### Create a Slack Webhook

1.  **Go to  [Slack API](https://api.slack.com/apps)**
    
2.  **Click "Create New App"**
    
3.  **Select "From scratch"**
    
4.  **Enter app name:**  "Kubernetes Alerts"
    
5.  **Choose your workspace**
    
6.  **Click "Incoming Webhooks"**  in the left sidebar
    
7.  **Toggle "Activate Incoming Webhooks" to ON**
    
8.  **Click "Add New Webhook to Workspace"**
    
9.  **Choose your alert channel**  (like #alerts or #dev-team)
    
10.  **Copy the webhook URL**  (starts with  `https://hooks.slack.com/`)
    

### Connect Slack to SigNoz

1.  **In SigNoz, go to Settings → Notification Channels**
    
2.  **Click "New Alert Channel"**
    
3.  **Fill in the details:**
    
    -   **Name:**  Kubernetes Alerts
        
    -   **Type:**  Slack
        
    -   **Webhook URL:**  [Paste your webhook URL]
        
    -   **Channel:**  #alerts (or your chosen channel)
        
4.  **Click "Test"**  - you should get a test message in Slack!
    
5.  **Click "Save"**    



## Step 7: Setting Up the Alert in SigNoz

### 7.1 Open the alert builder

1.  **Go to "Alerts" in SigNoz**
    
2.  **Click "New Alert Rule"**

3.   **Select Metrics-based Alert**
    
4.  **Configure the alert:**
    

### 7.2  Alert Details:

### Define the metric

-   **Metric name:** `k8s.pod.phase`
    
-   **Filter query:**
   
`phase = "Pending" AND k8s.cluster.name = "<CLUSTER_NAME>"`

(Replace `<CLUSTER_NAME>` with the name of your cluster )

-   **AGGREGATE WITHIN TIME SERIES:** set **Avg** and **every = 60 Seconds**.
    
-   **AGGREGATE ACROSS TIME SERIES:** set **Max** and **by = `k8s.namespace.name`, `k8s.pod.name`** (so each pod is evaluated separately).
    
Click **Stage & Run Query** and you should see 1 when a pod is Pending, 0 otherwise.

#### Define Alert Conditions

-   **Send a notification when**: `A` **is** `above` the threshold
    
-   **Alert Threshold:** `1`
    
-   **…during the last:** `5 mins`

-  Change **“at least once”** → **“all the times”**
    
-   (Optional) **Evaluate every:** `1 min` 
    
#### Name, message, and routing

14.  **Alert name:** `Pod stuck in Pending (>5m)`
    
15.  **Severity:** `Warning` _(or `Critical`, your policy)_
    
16.  **Description / Message (example):**
    
    `Pod {{k8s.pod.name}} in {{k8s.namespace.name}} has been Pending >5m on {{k8s.cluster.name}}.` 
    
17.  **Notification channels:** select your Slack channel (you added it in **Settings → Alert Channels**).
    
18.  Click **Save**.


## Step 8: Test

Apply a “can’t schedule” pod, for example:

``` yaml 
apiVersion: v1
kind: Pod
metadata:
  name: pending-demo
  namespace: default
spec:
  containers:
    - name: busybox
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
      resources:
        requests:
          cpu: "1000"
          memory: "1Ti"
 ```

Then:

`kubectl apply -f pending-pod.yaml`

`kubectl get pod pending-demo -w` 

After **~5 minutes**, you should get the Slack alert.

* _Once basic pod monitoring is established, consider implementing additional monitoring patterns for comprehensive Kubernetes observability._

## Step 9: Clean Up

1. **Turn off the alert (Disable or Delete), Clear Dashboards, & Remove Notification Channel** 
  
2. **Delete the test pod**

` kubectl delete -f k8s/pending-pod.yaml --ignore-not-found `

3. **Uninstall SigNoz K8s-Infra**

`helm uninstall my-release -n signoz || true`

`kubectl delete ns signoz --ignore-not-found`

4. **Rotate secrets (important)**

* In SigNoz Cloud: **manage the ingestion key** used in `override-values.yaml`. (Settings → Ingestion)
* In Slack: **revoke/rotate** the Incoming Webhook you created.

5. **Remove secrets file**

`rm -f k8s/override-values.yaml`

6. **Delete the demo cluster**

`minikube delete`


## FAQs

### Do I need to install OpenTelemetry separately?

**Short answer:** No.  
**Why:** The **SigNoz `k8s-infra` Helm chart** already deploys the OpenTelemetry Collector (agents + cluster collector) with the right receivers and exporters.  


----------

### I ran `helm install`, but I don’t see anything in the cluster

**Likely causes & fixes:**

-   Wrong namespace:
    
    `helm list -A `
    
    `kubectl get all -n signoz`
    
    Always install with `-n signoz --create-namespace` and check in the same namespace.
    
-   Bad path to values:
    
    ```bash
    helm upgrade --install my-release signoz/k8s-infra -n signoz -f override-values.yaml --debug --atomic
    ```
    
    `--debug --atomic` surfaces template/validation errors.
    
-   Nothing rendered (bad values keys):  
    Render locally to see what Helm would apply:
    
    ```bash
    helm template my-release signoz/k8s-infra -n signoz -f override-values.yaml | head -n 60 
    ```
    
----------

### Metrics aren’t showing up in SigNoz

**Checklist:**

-   Are the collector pods Running?
    
    ```bash
    kubectl get pods -n signoz
    ```
    
-   Endpoint & key correct in `override-values.yaml`?
    
    ```yaml
    otelCollectorEndpoint: ingest.<REGION>.signoz.cloud:443
    signozApiKey: <YOUR_KEY>
    otelInsecure: false
    ```
    
-   Ensure outbound HTTPS (443) is allowed.

---

For any queries, reach out via email or Slack.
