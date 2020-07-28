# Install Prometheus & integrate Trident's metrics

**GOAL:**  
Trident 20.01.1 introduced metrics that can be integrated into Prometheus.  
Going through this scenario at this point will be interesting as you will actually see the metrics evolve with all the labs.  

You can either follow this scenario or go through the following link:  
<https://netapp.io/2020/02/20/a-primer-on-prometheus-trident/>

## A. Install Helm

Helm, as a packaging tool, will be used to install Prometheus.

```bash
# cd
# wget https://get.helm.sh/helm-v3.0.3-linux-amd64.tar.gz
# tar xzvf helm-v3.0.3-linux-amd64.tar.gz
# cp linux-amd64/helm /usr/bin/
```

## B. Install Prometheus in its own namespace

```bash
# kubectl create namespace monitoring
# helm repo add stable https://kubernetes-charts.storage.googleapis.com
# helm install prom-operator stable/prometheus-operator  --namespace monitoring
```

You can check the installation with the following command:

```bash
# helm list -n monitoring
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                           APP VERSION
prom-operator   monitoring      1               2020-04-30 12:43:12.515947662 +0000 UTC deployed        prometheus-operator-8.13.4      0.38.1
```

## C. Expose Prometheus

Prometheus got installed pretty easily.
But how can you access from your browser?

The way Prometheus is installed required it to be access from the host where it is installed (with a *port-forwarding* mechanism for instance).
We will modify the Prometheus service in order to access it from anywhere in the lab, with why not a *NodePort* configuration

```bash
# kubectl edit -n monitoring svc prom-operator-prometheus-o-prometheus
```

### BEFORE

```bash
spec:
  clusterIP: 10.96.69.69
  ports:
  - name: web
    port: 9090
    protocol: TCP
    targetPort: 9090
  selector:
    app: prometheus
    prometheus: prom-operator-prometheus-o-prometheus
  sessionAffinity: None
  type: ClusterIP
```

### AFTER: (look at the ***nodePort*** & ***type*** lines)

```bash
spec:
  clusterIP: 10.96.69.69
  ports:
  - name: web
    port: 9090
    nodePort: 30000
    protocol: TCP
    targetPort: 9090
  selector:
    app: prometheus
    prometheus: prom-operator-prometheus-o-prometheus
  sessionAffinity: None
  type: NodePort
```

You can now access the Prometheus GUI from the browser using the port 30000 on RHEL5 address (<http://192.168.0.66:30000>)

## D. Add Trident to Prometheus

Refer to the blog aforementioned to get the details about how this Service Monitor works.
The following link is also a good place to find information:
<https://github.com/coreos/prometheus-operator/blob/master/Documentation/user-guides/getting-started.md>

In substance, we will tell in this object to look at services that have the label *trident* & retrieve metrics from its endpoint.
The Yaml file has been provided and is available in the Scenario2 sub-directory

```bash
# kubectl create -f /root/NetApp-LoD/trident_with_k8s/tasks/config_prometheus/Trident_ServiceMonitor.yml
servicemonitor.monitoring.coreos.com/trident-sm created
```

## E. Check the configuration

On the browser in the LoD, you can now connect to the address <http://192.168.0.66:30000> in order to access Prometheus
You can check that the Trident endpoint is taken into account & in the right state by going to the menu STATUS => TARGETS

![Trident Status in Prometheus](../../../images/Trident_status_in_prometheus.png "Trident Status in Prometheus")

If you don't see anything regarding Trident, please make sure you have also carried out the [Installing Trident task](../install_trident).

## F. Play around

Now that Trident is integrated into Prometheus, you can retrieve metrics or build graphs.

## G. What's next

Now that Trident is connected to Prometheus, you can move to the next task:  
- [Configure Grafana & add your first graphs](../config_grafana)    

or jump ahead to...
- [Configure your first NAS backends & storage classes](../config_file) 

---
**Page navigation**  
[Top of Page](#top) | [Home](/README.md) | [Full Task List](/README.md#dev-k8s-cluster-tasks)
