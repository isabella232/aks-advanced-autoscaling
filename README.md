# Welcome to: Module 4: Configure Keda Using Http Metrics & Open Service Mesh and Testing with Azure Load Testing

### Install OpenServiceMesh

* Execute the following
```
az aks enable-addons --addons open-service-mesh -g 'resource_group' -n 'aks_cluster-name'
```
* Verify it was enabled
```
az aks show -g 'resource_group' -n 'aks_cluster-name'  --query 'addonProfiles.openServiceMesh.enabled'
```

### Verify the status of OSM in kube-system namespace

* Execute the following

```
kubectl get deployments -n kube-system --selector app.kubernetes.io/name=openservicemesh.io
kubectl get pods -n kube-system --selector app.kubernetes.io/name=openservicemesh.io
kubectl get services -n kube-system --selector app.kubernetes.io/name=openservicemesh.io

```

### Installing Prometheus via helm chart kube-prometheus-stack

* Execute the following

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus \
prometheus-community/kube-prometheus-stack -f https://raw.githubusercontent.com/javierromancsa/aks-keda-http-osm-autoscaling/main/byo_values.yaml \
--namespace monitoring \
--create-namespace

```

### Disabled metrics scapping from components that AKS don't expose.

* Execute the following

```
helm upgrade prometheus \
prometheus-community/kube-prometheus-stack -f https://raw.githubusercontent.com/javierromancsa/aks-keda-http-osm-autoscaling/main/byo_values.yaml \
--namespace monitoring \
--set kubeEtcd.enabled=false \
--set kubeControllerManager.enabled=false \
--set kubeScheduler.enabled=false

```

### In OSM CLI 

* Execute the following 

```
OSM_VERSION=v0.11.1
curl -sL "https://github.com/openservicemesh/osm/releases/download/$OSM_VERSION/osm-$OSM_VERSION-linux-amd64.tar.gz" | tar -vxzf -
sudo mv ./linux-amd64/osm /usr/local/bin/osm
sudo chmod +x /usr/local/bin/osm

```

### Adding namespace to mesh and enabling OSM metrics
* Execute the following

```

osm namespace add order-portal order-processor
osm metrics enable --namespace "order-portal, order-processor"
sleep 5s
kubectl rollout restart deployment order-web -n order-portal

```

### Portforward Prometheus in another new terminal and open http://localhost:9090 :
```
kubectl port-forward svc/prometheus-kube-prometheus-prometheus -n monitoring 9090
```

### Query to run in Prometheus to pull http metrics

```
envoy_cluster_upstream_rq_xx{envoy_response_code_class="2"}
sum(rate(envoy_cluster_upstream_rq_xx{envoy_cluster_name="order-portal"}[1m]))
```

### Installing Contour in AKS:

link[https://projectcontour.io/getting-started/#option-2-helm]
```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install mycontour bitnami/contour --namespace projectcontour --create-namespace

kubectl -n projectcontour get po,svc
```

### Create HTTPProxy and ingressBackend for bookstore appplication
#### Get the public/External IP of the Azure loadbalancer created for the Contour Services
```
dns=".nip.io"
myip="$(kubectl -n projectcontour describe svc -l app.kubernetes.io/component=envoy | grep -w "LoadBalancer Ingress:"| sed 's/\(\([^[:blank:]]*\)[[:blank:]]*\)*/\2/')"
myip_dns=$myip$dns

```
#### Create HTTPProxy and ingressBackend
```
kubectl apply -f - <<EOF
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: orderportalproxy
  namespace: order-portal
spec:
  virtualhost:
    fqdn: $myip_dns
  routes:
  - services:
    - name: kedasampleweb
      port: 80
---
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: orderportalbackend
  namespace: order-portal
spec:
  backends:
  - name: kedasampleweb
    port:
      number: 80 # targetPort of httpbin service
      protocol: http
  sources:
  - kind: Service
    namespace: projectcontour
    name: mycontour-contour-envoy
EOF

```

### Create KEDA ScaledObject based on Query

```
kubectl apply -f https://raw.githubusercontent.com/javierromancsa/aks-keda-http-osm-autoscaling/main/keda_bookstore_http.yaml
```


### Create KEDA ScaledObject based on Query

```

```

