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
You have to change the Service Type from ClusterIP --> NodePort. Using your Master Node's IP with the exposed port of NodePort you can able to access the grafana server.


# When logging in, use username "admin" and get password by running the following:
kubectl get secret --namespace grafana grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```


# Creating Persitent Volume with AWS EC2 Volume:
Now we have to create one ec2 volume using below aws clid command. 
```
aws ec2 create-volume --availability-zone us-east-1a 

```
Now you have to same the volume ID somewhere. It will require while attaching pv

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv1
spec:
  storageClassName: gp2
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  awsElasticBlockStore:
    fsType: ext4
    volumeID: <volumeID>   # Put your volume ID here
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc1
spec:
  storageClassName: gp2
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

Create the PV and PVC using below kubectl commands
```
kubectl apply -f pv.yaml        # Creating PV
kubectl apply -f pvc.yaml       # Creating PVC
```


