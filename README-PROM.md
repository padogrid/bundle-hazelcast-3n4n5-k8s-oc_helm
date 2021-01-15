# Configuring Prometheus Metrics On OCP 4.5

- The project name, **oc-helm**, is used in this article. Make sure to replace it with your project name.
- Note that the label, `helm.sh/chart: hazelcast-enterprise-3.5.3`, specified in the `prometheuse/service-monitor.yaml` file is used to discover the Hazelcast metrics service. Make sure to change it to your Hazelcast metrics service label.

## 1. Enable service monitoring

```bash
oc -n openshift-monitoring edit configmap cluster-monitoring-config
```

Enter and save following:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    techPreviewUserWorkload:
      enabled: true
```

Check the `prometheus-user-workload` pods created.

```bash
oc -n openshift-user-workload-monitoring get pod
```

Output:

```console
NAME                                  READY   STATUS    RESTARTS   AGE
prometheus-operator-78755f496-pg6gj   2/2     Running   0          4h35m
prometheus-user-workload-0            5/5     Running   1          4h57m
prometheus-user-workload-1            5/5     Running   1          4h53m
thanos-ruler-user-workload-0          3/3     Running   0          4h57m
thanos-ruler-user-workload-1          3/3     Running   0          4h53m
```

## 2. Grant user permissions using web console

1. **User Management > Role Bindings > Create Binding**
2. In **Binding Type**, select the "Namespace Role Binding" type.
3. In **Name**, enter a name for the binding, e.g., "oc-helm-prometheus".
4. In **Namespace**, select the names space, e.g., "oc-helm".
5. In **Role Name**, enter **monitoring-edit**.
6. In **Subject**, select **User**.
7. In **Subject Name**, enter the name of the user, e.g., `dspark@sorintlab.com`.
8. Confirm the role binding.

## 3. Grant user permissions

```bash
oc policy add-role-to-user <role> <user> -n <namespace>

# Example:
oc policy add-role-to-user monitoring-edit dspark@sorintlab.com -n oc-helm
```

## 4. Setup metrics collection

First, check the label

```bash
oc describe svc oc-helm-hazelcast-enterprise-metrics
```

Output:

```console
Name:              oc-helm-hazelcast-enterprise-metrics
Namespace:         oc-helm
Labels:            app.kubernetes.io/instance=oc-helm
                   app.kubernetes.io/managed-by=Helm
                   app.kubernetes.io/name=hazelcast-enterprise
                   helm.sh/chart=hazelcast-enterprise-3.5.3
...
```

Edit `prometheus/service-monitor.yaml` and set the selector label, `helm.sh/chart`.

```bash
cd_k8s oc-helm
vi prometheus/service-monitor.yaml
```

For our example, set `helm.sh/chart: hazelcast-enterprise-3.5.3` at the end of the file as shown below.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    k8s-app: hazelcast-monitor
  name: hazelcast-monitor
spec:
  endpoints:
  - interval: 15s
    port: metrics
    scheme: http
  selector:
    matchLabels:
      helm.sh/chart: hazelcast-enterprise-3.5.3
```

Apply `prometheus/service-monitor.yaml`

```bash
oc apply -f prometheus/service-monitor.yaml
oc  get servicemonitor
```

```console
NAME                AGE
hazelcast-monitor   2m28s
```

## 5. Start Hazelcast

Start Hazelcast. See instructions in [bundle-hazelcast-4-k8s-oc_helm](README.md).

:exclamation: Hazelcast must be started after the above steps have been completed. Otherwise, you may not see Hazelcast metrics.

## 6. Access the metrics

### 6.1 As Administrator

1. From the web console, select **Monitoring > Metrics**
2. You can enter queries directly from the "Add Query" text fields.
3. Optionally, you can use the Prometheus UI by selecting the "Prometheus UI" link at the top.

### 6.2 As Developer

1. From the web console, select **Monitoring > Metrics**
2. Select **Show PromQL** and enter queries.

## 7. PromQL Exmaples:

### 7.1. Hazelcast cluster size (member count)

The member count or the cluster size can be monitored as follows.

```console
max(com_hazelcast_Metrics_size)
```

### 7.2. Memory/CPU

Memory and CPU usages can be monitored as shown below. For example, the percentage of the total used heap provides a single metric overview of the cluster health.

```console
# Total used heap percentage
sum(com_hazelcast_Metrics_usedHeap)/sum(com_hazelcast_Metrics_maxHeap)

# Max CPU- process (member) & OS
max(com_hazelcast_Metrics_processCpuLoad)
max(com_hazelcast_Metrics_systemCpuLoad)

# Max on-heap max
max(jvm_memory_bytes_max{area="heap"})

# Max on-heap used
max(jvm_memory_bytes_used{area="heap"})

# Max off-heap used
max(jvm_memory_bytes_used{area="nonheap"})
```

### 7.3. GC

Hazelcast should be run with G1 or CMS garbage colletor. The following metrics are available for both garbage collectors. Garbage collector specifics are shown in the subsequent sections. They are equivalent.

```console
# Hazelcast's Minor GC average milliseconds
com_hazelcast_Metrics_minorTime/com_hazelcast_Metrics_minorCount

# Hazelcast's Major GC average milliseconds
com_hazelcast_Metrics_majorTime/com_hazelcast_Metrics_majorCount
```

#### 7.3.1. G1

```console
# Young Generation - Minor GC average seconds
jvm_gc_collection_seconds_sum{gc="G1 Young Generation"}/jvm_gc_collection_seconds_count{gc="G1 Young Generation"}

# Old Generation - Major GC average seconds
jvm_gc_collection_seconds_sum{gc="G1 Old Generation"}/jvm_gc_collection_seconds_count{gc="G1 Oldmax(process_cpu_seconds_total)max(jvm_memory_bytes_used{area="nonheap"}) Generation"}
```

#### 7.3.2. CMS

```console
# Minor GC average seconds
jvm_gc_collection_seconds_sum{gc="PS Scavenge"}/jvm_gc_collection_seconds_count{gc="PS Scavenge"}

# Major GC average seconds
jvm_gc_collection_seconds_sum{gc="PS MarkSweep"}/jvm_gc_collection_seconds_count{gc="PS MarkSweep"}
```

### 7.4. Clients

```console
# Max client connections
max(com_hazelcast_Metrics_connectionListenerCount)

# The following seems to be always 0. Use the one above instead.
max(com_hazelcast_Metrics_clientCount)
```

### 7.5. File Descriptors

Hazelcast opens many file descriptors. If it reaches the max count then the Hazelcast member will become unresponsive.

```console
# Open FD count
com_hazelcast_Metrics_openFileDescriptorCount

# Max FD count
com_hazelcast_Metrics_maxFileDescriptorCount
```

### 7.6. Maps

You can monitor the number of entries in each map.

```console
# Max put count (eligibility and profile are IMaps)
max(com_hazelcast_Metrics_putCount{tag0="\"name=eligibility\""})
max(com_hazelcast_Metrics_putCount{tag0="\"name=profile\""})

# Max primary entry count
max(com_hazelcast_Metrics_ownedEntryCount{tag0="\"name=eligibility\""})
max(com_hazelcast_Metrics_ownedEntryCount{tag0="\"name=profile\""})

# Max backup entry count
max(com_hazelcast_Metrics_backupEntryCount{tag0="\"name=eligibility\""})
max(com_hazelcast_Metrics_backupEntryCount{tag0="\"name=profile\""})
```

## 8. List Monitoring URLs

```bash
oc -n openshift-monitoring get routes
```

Output:

```console
NAME                HOST/PORT                                                                            PATH   SERVICES            PORT    TERMINATION          WILDCARD
...
prometheus-k8s      prometheus-k8s-openshift-monitoring.apps.7919-681139.cor00005-2.cna.ukcloud.com             prometheus-k8s      web     reencrypt/Redirect   None
thanos-querier      thanos-querier-openshift-monitoring.apps.7919-681139.cor00005-2.cna.ukcloud.com             thanos-querier      web     reencrypt/Redirect   None
```

- Propmetheus: http://prometheus-k8s-openshift-monitoring.apps.7919-681139.cor00005-2.cna.ukcloud.com
- Prometheus metrics: http://prometheus-k8s-openshift-monitoring.apps.7919-681139.cor00005-2.cna.ukcloud.com/metrics
- Thanos: http://thanos-querier-openshift-monitoring.apps.7919-681139.cor00005-2.cna.ukcloud.com

## 9. Access Hazelcast metrics

Expose the metrics service.

```bash
oc get svc
```

Output:

```console
NAME                                     TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                        AGE
oc-helm-hazelcast-enterprise             ClusterIP      None             <none>        5701/TCP                       50m
oc-helm-hazelcast-enterprise-mancenter   LoadBalancer   172.30.2.240     <pending>     8080:30263/TCP,443:31951/TCP   50m
oc-helm-hazelcast-enterprise-metrics     ClusterIP      172.30.152.152   <none>        8080/TCP                       50m
```

```bash
oc expose svc oc-helm-hazelcast-enterprise-metrics
oc get route
```

Output:

```console
NAME                                   HOST/PORT                                                                                  PATH   SERVICES                               PORT      TERMINATION   WILDCARD
oc-helm-hazelcast-enterprise-metrics   oc-helm-hazelcast-enterprise-metrics-oc-helm.apps.7919-681139.cor00005-2.cna.ukcloud.com          oc-helm-hazelcast-enterprise-metrics   metrics                 None
```

- Hazelcast metrics: http://oc-helm-hazelcast-enterprise-metrics-oc-helm.apps.7919-681139.cor00005-2.cna.ukcloud.com

## 10. Teardown

```bash
# Stop Hazelcast first then delete the following.
oc delete -f prometheus/service-monitor.yaml
oc delete -f prometheus/custom-metrics-role.yaml
```

## References

1. Hazelcast OpenShift Helm Charts, [README.md](README.md).
2. Monitoring your own services, https://docs.openshift.com/container-platform/4.5/monitoring/monitoring-your-own-services.html
3. Prometheus Operator: Getting Started, https://github.com/openshift/prometheus-operator/blob/release-4.5/Documentation/user-guides/getting-started.md
4. Querying Prometheus, https://prometheus.io/docs/prometheus/latest/querying/basics/
