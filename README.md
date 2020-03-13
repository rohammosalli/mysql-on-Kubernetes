#### Part 1 
###### Setup Kubernetes Cluster on GKE

we can use Terraform to Deploy our GKE Cluster or just we can use Kubespray to deploy a self-hosted cluster

if you want to use Terraform to deploy GKE follow these steps 


Download your service account from GKE dashboard IAM

```bash
mkdir creds
cp DOWNLOADEDSERVICEKEY.json creds/serviceaccount.json
```
you need to create a provider.tf file 

vim provider.tf

```.yaml
provider "google" {
  credentials = "${file("./creds/serviceaccount.json")}"
  project     = "amir-251909"
  region      = "europe-north1"
  zone        = "europe-north1-a"

}
```
you need to create a gke-cluster.tf file 

vim gke-cluster.tf

```.yaml
resource "google_container_cluster" "gke-cluster" {
  name     = "standard-cluster-1"
  location = "europe-north1-a"
  remove_default_node_pool = true
  initial_node_count = 1
  timeouts {
    create = "30m"
    update = "20m"
  }
}

resource "google_container_node_pool" "primary_preemptible_nodes" {
  name       = "gke-node-pool"
  location   = "europe-north1-a"
  cluster    = "${google_container_cluster.gke-cluster.name}"
  node_count = 3
  timeouts {
    create = "30m"
    update = "20m"
  }
}
```
Then you can deploy cluster on GCP with ```terraform apply``` command
```bash
terraform plan

terraform apply
```
###### Self Hosted Cluster 

I used this link [High Available Kubernetes Cluster Setup using Kubespray](https://schoolofdevops.github.io/ultimate-kubernetes-bootcamp/cluster_setup_kubespray/) to deploy my cluster on my host 

 One node for Master & Worker for this task


Best Practice is 3 Master Node and 3 Worker Node at minimum 


###### Note :
1 - If you experience highly load traffic in your cluster you should take care of out of-resource handling in Kubelet, you have to reserve some memory and CPU for nodes because if you don't, the nodes in under pressure kernel will kill the process, so this means you will lose your node or even the cluster.

2 - If you use Master Node alos as worker Please don't do it in production 


If you want to deploy ingress controller you can use this way

```bash
helm install stable/nginx-ingress --namespace kube-system --name nginx  --set controller.hostNetwork=true,controller.kind=DaemonSet, --set controller.service.externalTrafficPolicy=Local
```

###### deploy promethus 

to deploy Prometheus I used help repo and just I changed some configs in values to config alert manager and Prometheus rules

```bash
helm upgrade --install monior  -f prometheus/values.yaml prometheus/
```

##### Config Alertmanger and Prometheus rules 

To config Alertmanaget better I added template and configured SMTP and Slack part that can send an alert via email and Slack channel  

```yaml

alertmanagerFiles:
  custom-template.tmpl: |
    {{ define "__alertmanager" }}Cluster: CLUSTER_NAME{{ end }}
    {{ define "__alertmanagerURL" }}{{ .ExternalURL }}/#/alerts?receiver={{ .Receiver }}{{ end }}
    {{ define "__subject" }}[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .GroupLabels.SortedPairs.Values | join " " }} {{ if gt (len .CommonLabels) (len .GroupLabels) }}({{ with .CommonLabels.Remove .GroupLabels.Names }}{{ .Values | join " " }}{{ end }}){{ end }}{{ end }}
    {{ define "__description" }}{{ end }}
    {{ define "__text_alert_list" }}{{ range . }}Labels:
    {{ range .Labels.SortedPairs }} - {{ .Name }} = {{ .Value }}
    {{ end }}Annotations:
    {{ range .Annotations.SortedPairs }} - {{ .Name }} = {{ .Value }}
    {{ end }}Source: {{ .GeneratorURL }}
    {{ end }}{{ end }}
    {{ define "slack.default.title" }}{{ template "__subject" . }}{{ end }}
    {{ define "slack.default.username" }}{{ template "__alertmanager" . }}{{ end }}
    {{ define "slack.default.fallback" }}{{ template "slack.default.title" . }} | {{ template "slack.default.titlelink" . }}{{ end }}
    {{ define "slack.default.pretext" }}{{ end }}
    {{ define "slack.default.titlelink" }}{{ template "__alertmanagerURL" . }}{{ end }}
    {{ define "slack.default.iconemoji" }}{{ end }}
    {{ define "slack.default.iconurl" }}{{ end }}
    {{ define "slack.default.text" }}{{ end }}
    {{ define "slack.default.footer" }}{{ end }}
  alertmanager.yml:
    global:
      smtp_smarthost: 'smtp-relay.example.com:587'
      smtp_from: 'alert@gmail.com'
      smtp_auth_username: 'alert@gmail.com'
      smtp_auth_password: 'QZ3AFO4yEIrmkU9Y'
      smtp_require_tls: true
    # The directory from which notification templates are read.
    templates:
    - /etc/config/custom-template.tmpl
    # The root route on which each incoming alert enters.
    route:
      # The labels by which incoming alerts are grouped together. For example,
      # multiple alerts coming in for cluster=A and alertname=LatencyHigh would
      # be batched into a single group.
      group_by: ['alertname', 'cluster', 'service']

      # When a new group of alerts is created by an incoming alert, wait at
      # least 'group_wait' to send the initial notification.
      # This way ensures that you get multiple alerts for the same group that start
      # firing shortly after another are batched together on the first
      # notification.
      group_wait: 3s

      # When the first notification was sent, wait 'group_interval' to send a batch
      # of new alerts that started firing for that group.
      group_interval: 5s

      # If an alert has successfully been sent, wait 'repeat_interval' to
      # resend them.
      repeat_interval: 60m

      # A default receiver
      receiver: alert-emailer

      # All the above attributes are inherited by all child routes and can
      # overwritten on each.

      # The child route trees.
      routes:
      - match:
          service: node
        receiver: alert-emailer

        routes:
        - match:
            severity: critical
          receiver: alert-emailer

      # This route handles all alerts coming from a database service. If there's
      # no team to handle it, it defaults to the DB team.
      - match:
          service: database
        receiver: alert-emailer

        routes:
        - match:
            severity: critical
          receiver: alert-emailer

    receivers:
    - name: alert-emailer
      email_configs:
      - to: 'rohammosalli@gmail.com'
        headers:
          subject: "You have firing alerts"
      slack_configs:
      - channel: '#alerts'
        send_resolved: true
        api_url: http://slack.com/hooks/swh8coyj3pnabxecscperbpfxw
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
        title: '{{ template "slack.default.title" . }}'
        title_link: '{{ template "slack.default.titlelink" . }}'
        pretext: '{{ .CommonAnnotations.summary }}'
        text: |-
          {{ range .Alerts }}
             *Alert:* {{ .Annotations.summary }} - `{{ .Labels.severity }}`
             *Description:* {{ .Annotations.description }}
             *Details:*
          {{ range .Labels.SortedPairs }} • *{{ .Name }}:* `{{ .Value }}`
          {{ end }}
          {{ end }}
        fallback: '{{ template "slack.default.fallback" . }}'
        icon_emoji: '{{ template "slack.default.iconemoji" . }}'
        icon_url: '{{ template "slack.default.iconurl" . }}'
```

for the rules, for example, I added Disk usage alert in values 

```yaml 

rules:
    groups:
     - name: alerting_rules
       rules:
         - alert: OutOfDiskSpace
           expr: node_filesystem_free_bytes{mountpoint ="/"} / node_filesystem_size_bytes{mountpoint ="/"} * 100 < 10
           for: 5m
           labels:
             severity: warning
           annotations:
             message: "Out of disk space (instance {{ $labels.kubernetes_node }})."
             summary: "Out of disk space (instance {{ $labels.kubernetes_node }})"
             description: "Disk is almost full (< 10% left)\n"
```


###### deploy Storage class for Persist mysql data 
 
There is a couple of ways to create storage using SAN storage or NFS but in this role, I used local storage, after creating storage class PV we need to create PVC for our database storage


###### note: 
you have to change key and value base on your cluster setup and node label in this part matchExpressions

```yaml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-storage
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /root/payever/data
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - parskube1
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    name: mysql-pv-claim
spec:
    storageClassName: local-storage
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 5Gi           
```

#### deploy mysql with helm chart 

I pull mysql chart and I did some change on it, then I pass persistence controller in helm to telling mysql please use this mysql-pv-claim pvc to store data


```bash
helm upgrade --install -f mysql/values.yaml  --recreate-pods  mysql --set persistence.existingClaim=mysql-pv-claim mysql/

```
to enable slow log in mysql I just added these in values.yaml

```yaml
configurationFiles:
  mysql.cnf: |-
    [mysqld]
    long_query_time=5
    slow_query_log=1
    slow_query_log_file=/var/log/mysql/slow.lo
    log_queries_not_using_indexes
```    


###### deploy Grafana 

to deploy grafana again I used helm repo, the you grap promethus svc adress for Grafana data source.

```bash
helm upgrade --install grafana -f grafana/values.yaml grafana/
```

you can use this address to access to Grafan 

grafana.r-sysadmin.net

user:admin
to get password please use this 

```bash
 kubectl get secret --namespace default grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
 ```

###### Export slow logs as metrics to prometheus server using telegraf

to make this happend I just added telegraf as sidecar in mysql deployment helm chart
```yaml
containers:
      - image: docker.io/library/telegraf:1.4
        volumeMounts:
        - name: log   
          mountPath: /var/log/mysql  
        - name: telegraf-config
          mountPath: /etc/telegraf/
        imagePullPolicy: IfNotPresent
        name: telegraf
        ports:
        - containerPort: 9273
          name: tel-prom-port
          protocol: TCP  
```            
and also we need to add this port in mysql svc blow the ports: 

```yaml
ports:
  - name: tel-prom-port
    port: 9273
    protocol: TCP
    targetPort: 9273 
```
and also we have to add some anotation for mysql svc to say prometheus please scrap data from this end point 

```yaml
annotations:
    prometheus.io/path: /metrics
    prometheus.io/port: "9273"
    prometheus.io/scrape: "true"
```    
To export logs as metrics in added this config as configmap in mysql templates folder 
```yaml
[global_tags]
    [agent]
      interval = "10s"
      round_interval = true
      metric_batch_size = 1000
      metric_buffer_limit = 10000
      collection_jitter = "0s"
      flush_interval = "10s"
      flush_jitter = "0s"
      precision = ""
      debug = false
      quiet = false
      logfile = ""
      hostname = ""
      omit_hostname = false
    [[outputs.prometheus_client]]
      listen = "0.0.0.0:9273"
      expiration_interval = "0"
    [[inputs.logparser]]
      files = ["/var/log/mysql/slow.log"]
      from_beginning = false
      [inputs.logparser.grok]
        patterns = ['# Query_time: %{NUMBER:query}\s+Lock_time: %{NUMBER:lock_wait} Rows_sent: %{NUMBER:row_sent}\s*Rows_examined: %{NUMBER:rows_examined}']
        measurement = "slow_logs"
```
then Prometheus should grab the data from and MySQL svc endpoint but I had some problems in this part and I couldn't fix it I really not recommended this way for extract metrics because every second we have a different query and if we will use this way it's better to save it on Influxdb instead off Prometheus because Telegraf has good integration with Influxdb and also it' better we pars logs and save it on Elastic search 


#### - What problems did you encounter? How did you solve them?
I had some problem I didn't have any idea how I want to export logs metrics because I didn't use telegraf before so I spend some time to understand what telegraf do even I didn't find any useful example so I did some practice by my self and I figure out I need to use telegraf as sidecar in MySQL deployment 


#### - How would you setup backups for mysql in production
Tehere is some way to setup backup from MySQL in production with Kubernetes:
 1 - Cron Job with some script to do that 
 2 - using Cron Job and mysqldump 
 3 - using Percona-based deployment and backup solution 

#### How would you setup failover of mysql for production in k8s? How many nodes?

there som way to do that frist of all we need 3 node at minimum and we can create mysql cluster or just 1 pod and cloud base storage like S3 with custom storage scheduler allows co-locating the pod on the exact node where the data is stored. It ensures that an appropriate node is selected for scheduling the pod.-

##### Note: 

I set all the password in helm values file's it's not secure way it's nice to configure your user and password as env in CICD proccess the pass it to to helm contriller with --set when you wnat run helm install or helm upgrade 