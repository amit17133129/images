# Kubernetes-AWS-Volume Monitoring Using prometheus and grafana

In this task i have an requiremnt to monitor the pv volumes using grafana and promtheus. 

We will install promtheus and grafana in kops cluster usingn helm. Below are the command to install helm.

![installing helm](https://github.com/amit17133129/images/blob/main/images/images2/1.png?raw=true)

## Helm Installation Commands:

```
wget https://get.helm.sh/helm-v3.6.0-linux-amd64.tar.gz 

tar xvf helm-v3.6.0-linux-amd64.tar.gz

sudo mv linux-amd64/helm /usr/local/bin

rm helm-v3.6.0-linux-amd64.tar.gz

helm version
```

## Prometheus Installation using Helm:

```
# Creating a prometheus namespace
kubectl create namespace prometheus

# Adding prometheus repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Installing prometheus using helm
helm upgrade -i prometheus prometheus-community/prometheus \
    --namespace prometheus \
    --set alertmanager.persistentVolume.storageClass="gp2",server.persistentVolume.storageClass="gp2"

# Check pods in prometeus namespace
kubectl get pods -n prometheus
```
You will see same output as below.

```

NAME                                             READY   STATUS    RESTARTS   AGE
prometheus-alertmanager-59b4c8c744-r7bgp         1/2     Running   0          48s
prometheus-kube-state-metrics-7cfd87cf99-jkz2f   1/1     Running   0          48s
prometheus-node-exporter-jcjqz                   1/1     Running   0          48s
prometheus-node-exporter-jxv2h                   1/1     Running   0          48s
prometheus-node-exporter-vbdks                   1/1     Running   0          48s
prometheus-pushgateway-76c444b68c-82tnw          1/1     Running   0          48s
prometheus-server-775957f748-mmht9               1/2     Running   0          48s
```

You can check this prometheis running in your local dashboard with the following command
```
kubectl --namespace=prometheus port-forward deploy/prometheus-server 9090
```



## Installing Grafana Using Helm: 

```
# Create Namespace of grafana using below command:
kubectl create namespace grafana

# Install grafana using below command
helm install grafana stable/grafana \
    --namespace grafana \
    --set persistence.storageClassName="gp2" \
    --set persistence.enabled=true \
    --set adminPassword='hellografana' \
    --set datasources."datasources\.yaml".apiVersion=1 \
    --set datasources."datasources\.yaml".datasources[0].name=Prometheus \
    --set datasources."datasources\.yaml".datasources[0].type=prometheus \
    --set datasources."datasources\.yaml".datasources[0].url=http://prometheus-server.prometheus.svc.cluster.local \
    --set datasources."datasources\.yaml".datasources[0].access=proxy \
    --set datasources."datasources\.yaml".datasources[0].isDefault=true \
    --set service.type=LoadBalancer

# Check if Grafana is deployed properly
kubectl get all -n grafana

# You will have to expose the grafana service using NodePort with the below command
kubectl edit svc grafan -n grafana
You have to change the Service Type from ClusterIP --> LoadBalancer. Using your Master Node's IP with the exposed port of LoadBalancer you can able to access the grafana server.


# When logging in, use username "admin" and get password by running the following:
kubectl get secret --namespace grafana grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```
You can check LoadBalancer Url with its respective ports 
![installing helm](https://github.com/amit17133129/images/blob/main/images/images2/4.png?raw=true)

# Creating Persitent Volume with AWS EC2 Volume:
Now we have to create one ec2 volume using below aws clid command. 
```
aws ec2 create-volume --availability-zone us-east-1a 

```
Now you have to same the volume ID somewhere. It will require while attaching pv

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Retain
mountOptions:
  - debug
volumeBindingMode: Immediate
```
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv1
spec:
  accessModes:
  - ReadWriteOnce
  awsElasticBlockStore:
    fsType: xfs
    volumeID: aws://us-east-1a/<volumeID>   # Put your volume ID here
  capacity:
    storage: 10Gi
  persistentVolumeReclaimPolicy: Retain
  storageClassName: gp2-retain
  volumeMode: Filesystem

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  labels:
    app: mysql
  name: pvc1
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: gp2-retain
  volumeMode: Filesystem
  volumeName: pv1
```

Create the PV and PVC using below kubectl commands
```
kubectl apply -f pv.yaml        # Creating PV
kubectl apply -f pvc.yaml       # Creating PVC
```

Now we have to create pod and that pod will use the volume which we have created on aws.
```
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: centos
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo $(date -u) >> /data/out.txt; sleep 5; done"]     #it will write the data of date in mounted data folder
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: pvc1
```
You can run the following command to create the pod.
```
kubectl apply -f pod.yaml
```

Now when you switched to grafana and search for `pvc` in the metrics and then execute theose metrics then it will tell you the usage of the aws ebs volume.

![installing helm](https://github.com/amit17133129/images/blob/main/images/images2/2.png?raw=true)

![installing helm](https://github.com/amit17133129/images/blob/main/images/images2/3.png?raw=true)

# Monitoring Ec2 Volumes Using AlertManager

Here i am using alertmanager to send the alerts monitor the logs of the ec2 volumes using cloud-watch-exporter.

You can install using below command.
```
# Installing Cloudwatch Exporter
helm install cloudwatch  prometheus-community/prometheus-cloudwatch-exporter

# Checking logs of Cloud-Watch exporter
kubectl logs -f cloudwatch-prometheus-cloudwatch-exporter-545f78f77-kgrqg
```

After this you have to set the rules in the cloudwatch exporter for lack notifications. 
```
Slack Notification
```

Make sure that you calling the alertmanager server and its rules associated with it from the prometheus.yaml file. 
You can go insie prometheus.yaml file and then you can add below code to enable the alert-manager and its rules.

```
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093    

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  - rules.yml"
```
Here how rules.yml file look alike.
```
# Rule for KubernetesVolumeOutOfDiskSpace
- alert: KubernetesVolumeOutOfDiskSpace
    expr: kubelet_volume_stats_available_bytes / kubelet_volume_stats_capacity_bytes * 100 < 20
    for: 1m
    labels:
      severity: warning
  annotations:
         summary: Kubernetes Volume out of disk space (instance {{ $labels.instance }})
         description: "Volume is almost full (< 20% left)\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
```

Now we have alert manager prometheus configuration  ready. We will update the respective changes in the alertmanager server.

We need to have rule for slack noth=fications in respective channel. Add below code in alertmanager.yml file in the alertmanager server.
```

global:
      resolve_timeout: 5m
      slack_api_url: "enter_your_slack_webhook_url_here"
route:
      group_by: ['job']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: 'null'
      routes:
      - match:
          alertname: DeadMansSwitch
        receiver: 'null'
      - match:
        receiver: 'slack'
        continue: true
receivers:
    - name: 'null'
    - name: 'slack'
      slack_configs:
      - channel: 'alertmgr-stg-alerts'
        send_resolved: true
        title: '[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] Monitoring Event Notification'
        text: >-
          {{ range .Alerts }}
            Alert: {{ .Annotations.summary }} - `{{ .Labels.severity }}`
            Description: {{ .Annotations.description }}
            Graph: <{{ .GeneratorURL }}|> Runbook: <{{ .Annotations.runbook }}|>
            Details:
            {{ range .Labels.SortedPairs }} • {{ .Name }}: `{{ .Value }}`
            {{ end }}
          {{ end }}
templates:
    - '/etc/alertmanager/config/*.tmpl'
```
Now you will be able to see the graphs of alertmanger in the grafana dashboard. You can serach for alertmanager in the dashboard and you will able to see the below graphs.

![installing helm](https://github.com/amit17133129/images/blob/main/images/images2/6.png?raw=true)


![installing helm](https://github.com/amit17133129/images/blob/main/images/images2/7.png?raw=true)
