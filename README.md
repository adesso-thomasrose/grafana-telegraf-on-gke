# Grafana with telegraf using Kubernetes on Google Cloud Platform

A simple demonstration of running InfluxDB and Grafana using Kubernetes on Google Cloud Platform. Persistent disks are used for both InfluxDB and Grafana. Also demonstrates running Telegraf on every node in the cluster for monitoring.

Within the Kubernetes cluster InfluxDB will be accessible at `http://influxdb:8086`.

## Setup Grafana
Create a persistent disk to be used for storing Grafana state:
```
gcloud compute disks create grafana --size=50GB --zone=europe-west1-c
```
and check:
```
$ gcloud compute disks list grafana
NAME     ZONE            SIZE_GB  TYPE         STATUS
grafana  europe-west1-c  50       pd-standard  READY
```
Create a persistent volume & claim:
```
kubectl create -f pv/grafana.yaml
kubectl create -f pvc/grafana.yaml
```
Create the Grafana deployment:
```
kubectl create -f deployments/grafana.yaml
```
and wait for the pod to enter the running state:
```
$ kubectl get pods
NAME                       READY     STATUS    RESTARTS   AGE
grafana-2990892256-va1h5   1/1       Running   0          12s
influxdb-967644454-gwfly   1/1       Running   0          9m
```
Create the Grafana service:
```
kubectl create -f services/grafana.yaml
```
A quick way to check that Grafana is working is to first create a proxy
```
$ kubectl proxy
Starting to serve on 127.0.0.1:8001
```
then the Grafana UI should be visible in your web browser at <http://localhost:8001/api/v1/proxy/namespaces/default/services/grafana/>

## Setup Telegraf
Create a database, a user which can write into the database, and another use which can read from the database:
```
$ kubectl exec influxdb-967644454-gwfly -i -t -- bash -il
root@influxdb-967644454-gwfly:/# export TERM=vt100
root@influxdb-967644454-gwfly:/# influx
Visit https://enterprise.influxdata.com to register for updates, InfluxDB server management, and monitoring.
Connected to http://localhost:8086 version 1.1.0
InfluxDB shell version: 1.1.0
> auth
username: root
password:
> create database cluster
> CREATE USER telegraf WITH PASSWORD 'metrics'
> GRANT WRITE ON cluster TO telegraf
> CREATE USER reader WITH PASSWORD 'metrics'
> GRANT READ ON cluster TO reader
> exit
root@influxdb-967644454-gwfly:/# exit
```
We can use a daemonset to run instances of Telegraf on every node in the cluster. Firstly, create a configmap containing the Telegraf configuration file.
```
$ kubectl create configmap telegraf --from-file=config/telegraf.conf
```
This allows us to use a custom configuration file with the official Telegraf image. Then create the daemonset:
```
kubectl create -f daemonsets/telegraf.yaml
```
After a few moments there should be a Telegraf pod running on every node in the cluster (in this case 3):
```
$ kubectl get pods
NAME                       READY     STATUS    RESTARTS   AGE
grafana-2990892256-va1h5   1/1       Running   0          34m
influxdb-967644454-gwfly   1/1       Running   0          43m
telegraf-cfili             1/1       Running   0          30s
telegraf-rwsjo             1/1       Running   0          30s
telegraf-x34n8             1/1       Running   0          30s
```
Check that Telegraf is working:
```
$ kubectl logs telegraf-cfili
2016/12/02 22:50:33 I! Using config file: /etc/telegraf/telegraf.conf
2016/12/02 22:50:33 I! Starting Telegraf (version 1.1.1)
2016/12/02 22:50:33 I! Loaded outputs: influxdb
2016/12/02 22:50:33 I! Loaded inputs: inputs.disk inputs.diskio inputs.mem inputs.processes inputs.swap inputs.system inputs.net inputs.cpu
2016/12/02 22:50:33 I! Tags enabled: host=telegraf-cfili
2016/12/02 22:50:33 I! Agent Config: Interval:1m0s, Quiet:false, Hostname:"telegraf-cfili", Flush Interval:10s
HEPMBP095:influxdb-grafana-on-gke-master andrew$ kubectl logs telegraf-cfili
2016/12/02 22:50:33 I! Using config file: /etc/telegraf/telegraf.conf
2016/12/02 22:50:33 I! Starting Telegraf (version 1.1.1)
2016/12/02 22:50:33 I! Loaded outputs: influxdb
2016/12/02 22:50:33 I! Loaded inputs: inputs.disk inputs.diskio inputs.mem inputs.processes inputs.swap inputs.system inputs.net inputs.cpu
2016/12/02 22:50:33 I! Tags enabled: host=telegraf-cfili
2016/12/02 22:50:33 I! Agent Config: Interval:1m0s, Quiet:false, Hostname:"telegraf-cfili", Flush Interval:10s
2016/12/02 22:51:10 I! Output [influxdb] buffer fullness: 28 / 10000 metrics. Total gathered metrics: 28. Total dropped metrics: 0.
2016/12/02 22:51:10 I! Output [influxdb] wrote batch of 28 metrics in 8.803928ms
```
Try increasing the size of the cluster, this example from 3 to 4:
```
$ gcloud container clusters resize cluster-1 --size 4
```
We see that the size of the cluster has increased after a little while:
```
$ kubectl get nodes
NAME                                       STATUS    AGE
gke-cluster-1-default-pool-e5cf0a51-2ksi   Ready     1h
gke-cluster-1-default-pool-e5cf0a51-dkul   Ready     1h
gke-cluster-1-default-pool-e5cf0a51-f1sq   Ready     1h
gke-cluster-1-default-pool-e5cf0a51-sbg6   Ready     34s
```
and we now have 4 instances of Telegraf running:
```
$ kubectl get pods
NAME                       READY     STATUS    RESTARTS   AGE
grafana-2990892256-va1h5   1/1       Running   0          43m
telegraf-cfili             1/1       Running   0          10m
telegraf-cwjd6             1/1       Running   0          1m
telegraf-rwsjo             1/1       Running   0          10m
telegraf-x34n8             1/1       Running   0          10m
```
