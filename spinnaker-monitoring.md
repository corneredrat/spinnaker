# Monitoring spinnaker deployment in kubernetes

### Preconditions
This document assumes 
- *spinnaker deployment* is done in *spinnaker namespace*.
- Monitoring stack is to be deployed in the same kubernetes cluster.

# Steps
## Deploy monitoring agents
Monitoring agents needs to be running along with all spinnaker microservices. halyard provides commands to achieve this easily.

```bash
hal config metric-stores prometheus enable
hal deploy apply
```

Verify if the agents are deployed along with spinnaker microservices.
```bash
kubectl get pods -n spinnaker
```
### Installing Prometheus and Grafana

We will be using [kube-prometheus](github.com/coreos/kube-prometheus/) to deploy the monitoring stack. 

By default, it does not monitor namespaces other than kube-system and default. To monitor spinnaker running in its own namespace, we need to follow these steps :

```bash
# Install golang
wget https://dl.google.com/go/go1.13.3.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.13.3.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin #add it in bashrc for persistance

# Install jsonnet bundler
mkdir go
export GOPATH=$HOME/go
go get github.com/jsonnet-bundler/jsonnet-bundler/cmd/jb
export PATH=$PATH:/$HOME/go/bin
mkdir prometheus && cd prometheus
jb init
jb install github.com/coreos/kube-prometheus/jsonnet/kube-prometheus@release-0.1

# intall jsonnet
cd ~
git clone https://github.com/google/jsonnet
cd jsonnet/
make
sudo mv ./jsonnet /usr/local/bin

# install jsontoyaml
go get github.com/brancz/gojsontoyaml
```
Then, create two files in prometheus directory - `namespace.jsonnet` and `build.sh`


**namespace.jsonnet**
```
local kp = (import 'kube-prometheus/kube-prometheus.libsonnet') + {
  _config+:: {
    namespace: 'monitoring',

    prometheus+:: {
      namespaces+: ['spinnaker', 'logging'],
    },
  },
};

{ ['00namespace-' + name]: kp.kubePrometheus[name] for name in std.objectFields(kp.kubePrometheus) } +
{ ['0prometheus-operator-' + name]: kp.prometheusOperator[name] for name in std.objectFields(kp.prometheusOperator) } +
{ ['node-exporter-' + name]: kp.nodeExporter[name] for name in std.objectFields(kp.nodeExporter) } +
{ ['kube-state-metrics-' + name]: kp.kubeStateMetrics[name] for name in std.objectFields(kp.kubeStateMetrics) } +
{ ['alertmanager-' + name]: kp.alertmanager[name] for name in std.objectFields(kp.alertmanager) } +
{ ['prometheus-' + name]: kp.prometheus[name] for name in std.objectFields(kp.prometheus) } +
{ ['grafana-' + name]: kp.grafana[name] for name in std.objectFields(kp.grafana) }
```

**build.sh**
```bash
#!/usr/bin/env bash

# This script uses arg $1 (name of *.jsonnet file to use) to generate the manifests/*.yaml files.

set -e
set -x
# only exit with zero if all commands of the pipeline exit successfully
set -o pipefail

# Make sure to start with a clean 'manifests' dir
rm -rf manifests
mkdir manifests

                                               # optional, but we would like to generate yaml, not json
jsonnet -J vendor -m manifests "${1-example.jsonnet}" | xargs -I{} sh -c 'cat {} | gojsontoyaml > {}.yaml; rm -f {}' -- {}
```

Create manifest files:
```bash
chmod 777 build.sh
./build.sh namespace.jsonnet
```
Apply manifest files to your cluster

```bash
# enure you have admin rights on this cluster before executing the command
kubectl apply -f manifests
```

## ServiceMonitor and ConfigMaps

Create service monitor and configmaps in spinnaker namespace.

```bash
git clone https://github.com/spinnaker/spinnaker-monitoring.git

cd spinnaker-monitoring/spinnaker-monitoring-third-party/third_party/prometheus_operator/

# Below arguement "spinnaker" stands for namespace
./setup.sh spinnaker
```

## Generate dashboards

Get services running in monitoring namespace
```
kubectl get svc -n monitoring

NAME                    TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)             AGE
alertmanager-main       ClusterIP   10.20.7.180   <none>        9093/TCP            12m
alertmanager-operated   ClusterIP   None          <none>        9093/TCP,6783/TCP   12m
grafana                 ClusterIP   10.20.7.110   <none>        3000/TCP            12m
kube-state-metrics      ClusterIP   None          <none>        8443/TCP,9443/TCP   12m
node-exporter           ClusterIP   None          <none>        9100/TCP            12m
prometheus-k8s          ClusterIP   10.20.12.0    <none>        9090/TCP            12m
prometheus-operated     ClusterIP   None          <none>        9090/TCP            12m
prometheus-operator     ClusterIP   None          <none>        8080/TCP            12m

```
### Create tunnels for grafana and prometheus 
Because the scripts that we are about to run assumes these applications run in localhost by default.

I would recommend tmux to have these tunnels
```bash
tmux
# press [ctrl + b] and then "
kubectl port-forward svc/prometheus-k8s 9090:9090 -n monitoring

# press [ctrl + b] and then <down arrow>
kubectl port-forward svc/grafana 3000:3000 -n monitoring 

# press [ctrl + b] and "d" to minimize tmux
```

### Install scripts 
```bash
sudo apt-get update -y
sudo apt-get install spinnaker-monitoring-third-party
```

### Generate dashboards
```
/opt/spinnaker-monitoring/third_party/prometheus/install.sh --dashboards_only
```

## Expose grafana and prometheus to internet

### Edit grafana service

```
 kubectl edit svc grafana -n monitoring
```
Here, change -
- `port` : `3000` to `80` 
- `targetport` : `http` to `3000`
- `type` : `ClusterIP` to `LoadBalancer`

save and quit

### Edit grafana service

```
 kubectl edit svc prometheus-k8s -n monitoring
```
Here, change -
- `port` : `9090` to `80` 
- `targetport` : `http` to `9090`
- `type` : `ClusterIP` to `LoadBalancer`

save and quit

### Get public IPs of both grafana and prometheus

```bash
kubectl get svc -n monitoring
```

### Login to grafana
go to your browser, enter the IP address of grafana in your browser. By default, `admin` is the username and `admin` is the password. 

- Go to settings -> datasources. 
- Go prometheus (default)
- Modify `URL` to prometheus's Public IP. Make sure you remove port as well, since it is mapped to 80.

## Proceed to modify and view dashboards

# Further steps
- secure prometheus and grafana


