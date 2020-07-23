#########################################################################################
# SCENARIO 3: Configure Grafana & make your first graph
#########################################################################################

**GOAL:**  
Prometheus does not allow you to create a graph with different metrics, you need to use Grafana for that.  
Installing Prometheus with Helm also comes with this tool.  
We will learn how to access Grafana, and configure a graph.


## A. Expose Grafana

With Grafana, we are facing the same issue than with Prometheus with regards to accessing it.
We will then modify its service in order to access it from anywhere in the lab, with a *NodePort* configuration
```
# kubectl edit -n monitoring svc prom-operator-grafana
```


### BEFORE:
```
spec:
  clusterIP: 10.97.208.231
  ports:
  - name: service
    port: 80
    protocol: TCP
    targetPort: 3000
  selector:
    app.kubernetes.io/instance: prom-operator
    app.kubernetes.io/name: grafana
  sessionAffinity: None
  type: ClusterIP


```

### AFTER: (look at the ***nodePort*** & ***type*** lines)
```
spec:
  clusterIP: 10.97.208.231
  ports:
  - name: service
    nodePort: 30001
    port: 80
    protocol: TCP
    targetPort: 3000
  selector:
    app.kubernetes.io/instance: prom-operator
    app.kubernetes.io/name: grafana
  sessionAffinity: None
  type: NodePort
```

You can now access the Grafana GUI from the browser using the port 30001 on RHEL3 address (http://192.168.0.63:30001)


## B. Log in Grafana

### Accessing Grafana

---

The first time to enter Grafana, you are requested to login with a username & a password... But how does one find out what they are?  
Let's find the grafana pod and have a look at the pod definition, maybe there is a hint for us...

```bash
[root@rhel3 ~]# kubectl get pod -n monitoring -l app.kubernetes.io/name=grafana
NAME                                     READY   STATUS    RESTARTS   AGE
prom-operator-grafana-5dd648d5bc-2g6dn   3/3     Running   0          7h40m

[root@rhel3 ~]# kubectl describe pod prom-operator-grafana-5dd648d5bc-2g6dn -n monitoring
...
 Environment:
      GF_SECURITY_ADMIN_USER:      <set to the key 'admin-user' in secret 'prom-operator-grafana'>      Optional: false
      GF_SECURITY_ADMIN_PASSWORD:  <set to the key 'admin-password' in secret 'prom-operator-grafana'>  Optional: false
...
```

Let's see what grafana secrets there are in this cluster:

```bash
[root@rhel3 ~]# kubectl get secrets -n monitoring -l app.kubernetes.io/name=grafana
NAME                    TYPE     DATA   AGE
prom-operator-grafana   Opaque   3      7h50m

[root@rhel3 ~]# kubectl describe secrets -n monitoring prom-operator-grafana
Name:         prom-operator-grafana
...
Data
====
admin-user:      5 bytes
admin-password:  5 bytes
...
```

OK, so the data is there, but its encrypted... However, the admin can retrieve this information:

```bash
[root@rhel3 ~]# kubectl get secret -n monitoring prom-operator-grafana -o jsonpath="{.data.admin-user}" | base64 --decode ; echo
admin
[root@rhel3 ~]# kubectl get secret -n monitoring prom-operator-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
prom-operator
[root@rhel3 ~]#
```

Now we have the necessary clear text credentials to login to the Grafana UI at <http://192.168.0.141>.

---

The first time to enter Grafana, you are requested to login with a username & a password ...
But how to find out what they are ??

Let's look at the pod definition, maybe there is a hint there...

```
# kubectl get pod -n monitoring -l app.kubernetes.io/name=grafana
NAME                                     READY   STATUS    RESTARTS   AGE
prom-operator-grafana-7d99d7985c-98qcr   3/3     Running   0          2d23h

# kubectl describe pod prom-operator-grafana-7d99d7985c-98qcr -n monitoring
...
    Environment:
      GF_SECURITY_ADMIN_USER:      <set to the key 'admin-user' in secret 'prom-operator-grafana'>      Optional: false
      GF_SECURITY_ADMIN_PASSWORD:  <set to the key 'admin-password' in secret 'prom-operator-grafana'>  Optional: false
...
```
Let's check what secrets there are in this cluster
```
# kubectl get secrets -n monitoring -l app.kubernetes.io/name=grafana
NAME                    TYPE     DATA   AGE
prom-operator-grafana   Opaque   3      2d23h


# kubectl describe secrets -n monitoring prom-operator-grafana
Name:         prom-operator-grafana
...
Data
====
admin-password:  13 bytes
admin-user:      5 bytes
...
```
OK, so the data is there, and is encrypted... However, the admin can retrieve this information
```
# kubectl get secret -n monitoring prom-operator-grafana -o jsonpath="{.data.admin-user}" | base64 --decode ; echo
admin

# kubectl get secret -n monitoring prom-operator-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
prom-operator
```

There you go!
You can now properly login to Grafana.


## E. Configure Grafana

The first step is to tell Grafana where to get data (ie Data Sources).
In our case, the data source is Prometheus. In its configuration, you then need to put Prometheus's URL (http://192.168.0.63:30000)
You can also specify in this lab that Prometheus will be the default source.
Click on 'Save & Test'.


## F. Create your own graph

Hover on the '+' on left side of the screen, then 'New Dashboard', 'New Panel' & 'Add Query'.
You can here configure a new graph by adding metrics. By typing 'trident' in the 'Metrics' box, you will see all metrics available.


## G. Import a graph

There are several ways to bring dashboards into Grafana.  

*Manual Import*  
Hover on the '+' on left side of the screen, then 'New Dashboard' & 'Import'.
Copy & paste the content of the _Trident_Dashboard_Std.json_ file in this directory.  
The _issue_ with this method is that if the Grafana POD restarts, the dashboard will be lost...  

*Persistent Dashboard*  
The idea here would be to create a ConfigMap pointing to the Trident dashboard json file.

:mag:  
*A* **ConfigMap** *is an API object used to store non-confidential data in key-value pairs. Pods can consume ConfigMaps as environment variables, command-line arguments, or as configuration files in a volume. A ConfigMap allows you to decouple environment-specific configuration from your container images, so that your applications are easily portable.*  
:mag_right:  
```
# kubectl create configmap -n monitoring tridentdashboard --from-file=Trident_Dashboard_Std.json
configmap/tridentdashboard created

# kubectl label configmap -n monitoring tridentdashboard grafana_dashboard=1
configmap/tridentdashboard labeled
```
When Grafana starts, it will automatically load every configmap that has the label _grafana_dashboard_.  
In the Grafana UI, you will find the dashboard in its own _Trident_ folder.  

Now, where can you find this dashboard:  
- Hover on the 'Dashboard' icon on the left side bar (it looks like 4 small squares)  
- Click on the 'Manage' button  
- You then access a list of dashboards. You can either research 'Trident' or find the link be at the bottom of the page  

![Trident Dashboard](../../../images/trident_dashboard.jpg "Trident Dashboard")


## H. What's next

OK, you have everything to monitor Trident, let's continue with the creation of some backends :  
- [Scenario04](../Scenario04): Configure your first NAS backends & storage classes  
- [Scenario06](../Scenario06): Configure your first iSCSI backends & storage classes  
- [Scenario11](../Scenario11): Using Virtual Storage Pools  
- [Scenario15](../Scenario15): Dynamic export policy management  

---
**Page navigation**  
[Top of Page](#top) | [Home](/README.md) | [Full Task List](/README.md#prod-k8s-cluster-tasks)
