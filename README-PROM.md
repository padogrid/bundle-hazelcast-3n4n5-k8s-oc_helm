# Configuring Prometheus Metrics On OCP 4.5

- The project name, **oc-helm**, is used in this article. Make sure to replace it with your project name.
- Note that the label, `helm.sh/chart: hazelcast-enterprise-3.4.10`, specified in the `prometheuse/service-monitor.yaml` file is used to discover the Hazelcast metrics service. Make sure to change it to your Hazelcast metrics service label.

## Enable service monitoring

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

## Hazelcast

Start Hazelcast.

## Grant user permissions using web console

1. **User Management > Role Bindings > Create Binding**
2. In **Binding Type**, select the "Namespace Role Binding" type.
3. In **Name**, enter a name for the binding.
4. In **Namespace**, select the names space.
5. In **Role Name**, enter **monitoring-edit**.
6. In **Subject**, select **User**.
7. In **Subject Name**, enter the name of the user, i.e., `dspark@sorintlab.com`.
8. Confirm the role binding.

## Grant user permissions

```bash
oc policy add-role-to-user <role> <user> -n <namespace>

# Example:
oc policy add-role-to-user monitoring-edit dspark@sorintlab.com -n oc-helm
```

## Setup metrics collection

Apply `prometheus/service-monitor.yaml`

```bash
oc apply -f prometheus/service-monitor.yaml
oc  get servicemonitor
```

```console
NAME                AGE
hazelcast-monitor   2m28s
```

## Access the metrics as a developer

1. From the web console, select **Monitoring > Metrics**
2. Select **Show PromQL* and enter queries.

PromQL Exmaples:

### Hazelcast cluster size (member count)

```console
max(com_hazelcast_Metrics_size)
```

### Memory/CPU

```console
# Aggregate used heap sizes
sum(com_hazelcast_Metrics_usedHeap)

# Max CPU
max(process_cpu_seconds_total)

# Max on-heap max
max(jvm_memory_bytes_max{area="heap"})

# Max on-heap used
max(jvm_memory_bytes_used{area="heap"})

# Max off-heap used
max(jvm_memory_bytes_used{area="nonheap"})
```

### GC

```
# Minor GC average seconds
jvm_gc_collection_seconds_sum{gc="PS Scavenge"}/jvm_gc_collection_seconds_count{gc="PS Scavenge"}

# Major GC average seconds
jvm_gc_collection_seconds_sum{gc="PS MarkSweep"}/jvm_gc_collection_seconds_count{gc="PS MarkSweep"}
```

### Clients

```
# Max client connections
max(com_hazelcast_Metrics_clientCount)
```

### File descriptors

```
# Open FD count
com_hazelcast_Metrics_openFileDescriptorCount

# Max FD count
com_hazelcast_Metrics_maxFileDescriptorCount
```

### Maps

```
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

## Access all services as a cluster administrator

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

## Access Hazelcast metrics

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

## Teardown

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
