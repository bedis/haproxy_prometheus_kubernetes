# haproxy_prometheus_kubernetes
Use HAProxy in conjunction with Prometheus to feed Kubernetes Horizontal Pod Autoscaler

Please read carefuly the YAML files proposed here. There might be some adjustments to be peformed to make them match your environment.

## HAProxy Ingress Controller

### If Using HAProxy Enterprise ingress controller

```bash
kubectl create secret docker-registry haptech-registry \
	--docker-server=kubernetes-registry.haproxy.com \
	--docker-username=LOGIN \
	--docker-password=PASSWORD \
	--docker-email=EMAIL
```

Of course, replace **LOGIN**, **PASSWORD** and **EMAIL** with values relevent to you.

### Setup HAProxy ingress controller

* Generate a key / cert using your Kubernetes PKI and put them in a dir called **ca**
* Update Kube API server address in `kubeconfig.yml`
* Create the secret:
```bash
kubectl create secret generic hapee-kubeconfig \
	--from-file=./kubeconfig.yml \
	--from-file=./ca/ca.pem \
	--from-file=./ca/ingress-controller.pem \
	--from-file=./ca/ingress-controller.key
```
* Get a copy of **prometheus.lua** from [https://github.com/bedis/haproxy_lua_prometheus_exporter/blob/master/prometheus.lua](https://github.com/bedis/haproxy_lua_prometheus_exporter/blob/master/prometheus.lua) and put it into **hapee-ingress** directory
* Create the HAProxy configuration secret:
```bash
kubectl create secret generic hapee-configuration \
	--from-file=./hapee-ingress/haproxy.tmpl \
	--from-file=./hapee-ingress/prometheus.lua
```
* Set RBAC for the ingress controller:
```bash
kubectl create -f ingress-controller-rbac.yml
```
* Set config map for HAProxy's ingress controller:
```bash
kubectl create -f haproxy-ingress-configmap.yml
```
* Set HAProxy ingress controller Deployment: (adjust the image address if required)
```bash
kubectl create -f haproxy-ingress-de.yaml
```
* Set HAProxy ingress controller Service: (Note this also create a ServiceMonitor for prometheus-operator)
  (NOTE: don't forget to update the `clusterIP` and `nodePort` parameters when required)
```bash
kubectl create -f haproxy-ingress-svc.yaml
```

## Prometheus

* Install prometheus Operator:
```bash
kubectl create -f prometheus-operator.yaml
```
* Install prometheus Instance: (NOTE: don't forget to adjust `clusterIP`, `labels` and `selector` to match your environment)
```bash
kubectl create -f prometheus-instance.yaml
```
* Set up prometheus configuration for HAProxy: (Adjust the `expr` and `record` statements to match your environment)
```bash
RULES64=$(base64 -w0 metrics/prometheus_configuration/backend.rules)
PROM64=$(base64 -w0 metrics/prometheus_configuration/prometheus.yaml)
cat <<EOF >prometheus_configuration.yaml
apiVersion: v1
kind: Secret
data:
  backend.rules: ${RULES64}
  prometheus.yaml: ${PROM64}
metadata:
  name: prometheus-haproxy-metrics-prom
  namespace: default
```
* Install custom metrics endpoints in Kubernetes API service: (NOTE: update the `clusterIP` to match your environment)
```bash
kubectl create -f custom-metrics.yaml
```

## Kubernetes

* Finally, configure the Horizontal Pod Autoscaler (NOTE: update the application name and metrics accordingly to yoour environment)
```bash
kubectl create -f red-hpa.yaml
```

