### Create an Elastic Cloud deployment
You can use Elastic Cloud ( http://cloud.elastic.co ), or a local deployment.  whichever you choose, https://elastic.co/start will get you started.

If this is your first experience with the Elastic stack I would recommend Elastic Cloud; and don't worry, you do not need a credit card.

Make sure that you take note of the CLOUD ID and Elastic Password if you use Elastic Cloud or Elastic Cloud Enterprise.

### Authorization
Create a cluster level role binding so that you can manipulate the system level namespace

```
kubectl create clusterrolebinding cluster-admin-binding \
 --clusterrole=cluster-admin --user=<your email associated with the Cloud provider account>
```

### Clone the YAML files
Either clone the entire Elastic examples repo or use the wget commands in download.txt:

```
mkdir MonitoringKubernetes
cd MonitoringKubernetes
wget https://raw.githubusercontent.com/elastic/examples/master/MonitoringKubernetes/download.txt
sh download.txt
```

OR

```
git clone https://github.com/elastic/examples.git
cd examples/MonitoringKubernetes
```
### Set the credentials
Set these with the values from the http://cloud.elastic.co deployment

```
vi ELASTIC_PASSWORD
vi CLOUD_ID
```
and create a secret in the Kubernetes system level namespace

```
kubectl create secret generic dynamic-logging \
--from-file=./ELASTIC_PASSWORD --from-file=./CLOUD_ID \
--namespace=kube-system
```

### Create the cluster role binding for Metricbeat
```
kubectl create -f metricbeat-clusterrolebinding.yaml
```

### Check to see if kube-state-metrics is running
```
kubectl get pods --namespace=kube-system | grep kube-state
```
and create it if needed (by default it will not be there)

```
git clone https://github.com/kubernetes/kube-state-metrics.git kube-state-metrics
kubectl create -f kube-state-metrics/kubernetes
kubectl get pods --namespace=kube-system | grep kube-state
```

### Deploy the Guestbook example
Note: This is mostly the default Guestbook example from https://github.com/kubernetes/examples/blob/master/guestbook/all-in-one/guestbook-all-in-one.yaml

Changes:
 - added annotations so that Prometheus and Metricbeat would autodiscover the Redis pods
 - added an ingress that preserves source IPs
 - added ConfigMaps for the Apache2 and Mod-Status configs to block the /server-status endpoint from outside the internal network
 - added a redis.conf to set the slowlog time criteria

```
kubectl create -f guestbook.yaml
```
Let's look at a couple of things in guestbook.yaml:
 - Annotations on the Redis pods.  These will be used by both Prometheus and Metricbeat to autodiscover the Redis pods:
![Annotations](https://github.com/DanRoscigno/scraping-prometheus-k8s-with-metricbeat/blob/master/images/annotations.png)
 - Prometheus exporter for Redis sidecar:
 ![sidecar](https://github.com/DanRoscigno/scraping-prometheus-k8s-with-metricbeat/blob/master/images/sidecar.png)

### Verify the guestbook external IP is assigned

```
kubectl get service frontend -w
```
Once the external IP address is assigned you can type CTRL-C to stop watching for changes and get the command prompt back (the -w is "watch for changes")

### Deploy Metricbeat
Normally deploying Metricbeat would be a single command, but the goal of this example is to show multiple ways of pulling metrics from Prometheus, so we will do things step by step.

### Pull data from kube-state-metrics.
We will look specifically at the kubernetes event metricset when we build a visualization.  The event metricset exposes information about scaling deployments (among other things) and the reason for the scaling.

```
kubectl create -f metricbeat-kube-state-metrics.yaml
```
While that deploys, look at the snippet below.  You can see that Metricbeat will connect to port 8080 on the kube-state-metrics pod and collect events and state information about nodes, deployments, etc.
![kube-state-metrics YAML](https://github.com/DanRoscigno/scraping-prometheus-k8s-with-metricbeat/blob/master/images/kube-state-metrics.png)
### Pull data from the Prometheus exporter for Redis.
Up above is a screenshot of the YAML to deploy a sidecar to export Redis metrics.  Metricbeat can pull metrics from Prometheus exporters also.  Deploy a Metricbeat DaemonSet to autodiscover and collect these metrics.

Note: Normally the Metricbeat DaeomonSet would autodiscover and collect all of the metrics about the k8s environment and the apps running in there, this config is simplified to show just one example.
```
kubectl create -f metricbeat-prometheus-auto-discover.yaml
```
Let's look at how autodiscover is configured.  Earlier I showed the guestbook.yaml and that annotations were added to the Redis pods.  One of those annotations set prometheus.io/scrape to true, and the other set the port for the Redis metrics to 9121.  In the Metricbeat DaemonSet config we are configuring autodiscover to look for pods with the scrape and port annotations, which is exactly what Prometheus does.  
![Metricbeat autodiscover](https://github.com/DanRoscigno/scraping-prometheus-k8s-with-metricbeat/blob/master/images/metricbeat-autodiscover-exporters.png)
I made this autodiscover more abstract by not specifying port 9121, and substituting the value from the annotation provided by the k8s API so that a single autodiscover config could discover all exporters whether they are for Redis or another technology.

If you are not familiar with the Prometheus autodiscover configuration, here is part of an example.  Notice that it uses the same annotations:
![Promethus autodiscover](https://github.com/DanRoscigno/scraping-prometheus-k8s-with-metricbeat/blob/master/images/prometheus-autodiscover-snippet.png)

### Pull metrics from a Prometheus server
In this example we will pull:
 - self-monitoring metrics from the Prometheus server (using the /metrics endpoint)
 - all of the metrics that Prometheus collects from the various systems being monitored (using the /federate endpoint)

```
kubectl create -f metricbeat-prometheus-server.yaml
```
Here is the YAML to set this up:
![scrape server](https://github.com/DanRoscigno/scraping-prometheus-k8s-with-metricbeat/blob/master/images/metricbeat-prometheus-server.png)

### View in Kibana

Open your Kibana URL and look under the Dashboard link, verify that the Apache and Redis dashboards are populating.

### Scale your deployments and see new pods being monitored
List the existing deployments:
```
kubectl get deployments

NAME           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
frontend       3         3         3            3           3m
redis-master   1         1         1            1           3m
redis-slave    2         2         2            2           3m
```

Scale the frontend down to two pods:
```
kubectl scale --replicas=2 deployment/frontend

deployment "frontend" scaled
```

Check the frontend deployment:
```
kubectl get deployment frontend

NAME       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
frontend   2         2         2            2           5m
```

### View the changes in Kibana
See the screenshot, add the indicated filters and then add the columns to the view.  You can see the ScalingReplicaSet entry that is marked, following from there to the top of the list of events shows the image being pulled, the volumes mounted, the pod starting, etc.
![Kibana Discover](https://raw.githubusercontent.com/elastic/examples/master/MonitoringKubernetes/scaling-discover.png)
