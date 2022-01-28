# Kubernetes-Volume Monitoring Using prometheus and grafana

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
