# Hazelcast OpenShift Helm Charts

This bundle deploys Hazelcast using Helm Charts with Prometheus metrics enabled. It also includes the PadoGrid container for ingesting mock data into the Hazelcast cluster.

For Prometheus instructions plese see the following link: [Configuring Prometheus Metrics](README-PROM.md).

## Installing Bundle

```bash
install_bundle -download bundle-hazelcast-4-k8s-oc_helm
```

## Use Case

This bundle installs PadoGrid and Hazelcast Kubernetes containers to run on CodeReady Container (CRC) or OpenShift Container Platform (OCP). It demonstrates how to start Hazelcast using Helm Charts and  use the PadoGrid pod to ingest mock data into Hazelcast.

![OC Helm Charts Diagram](images/oc-helm.jpg)

## Required Software

- PadoGrid 0.9.3-SNAPSHOT+ (09/06/2020)
- OpenShift Client, **oc**
- [Helm](https://helm.sh/docs/intro/install/), **helm**

## Directory Tree View

```console
oc_helm/
├── bin_sh
│   ├── build_app
│   ├── cleanup
│   ├── login_padogrid_pod
│   ├── setenv.sh
│   ├── start_hazelcast
│   ├── start_padogrid
│   ├── stop_hazelcast
│   └── stop_padogrid
├── etc
│   └── hazelcast-enterprise
│       └── secret.yaml
├── hazelcast
│   └── values.yaml
└── padogrid
│   ├── padogrid.yaml
│   └── pv-hostPath.yaml
└── prometheus
    └── service-monitor.yaml
```

## 1. Build Local Environment

Run `build_app` which initializes your local environment. This script sets the license key in the `hazelcast/secret.yaml` file.

```bash
cd_k8s oc_helm; cd bin_sh
./build_app
```
### Changing Container Versions

The conatiner image versions can be changed as needed in the files shown below.

```bash
# Change dir to the k8s installation directory
cd_k8s oc_helm
```

| Container                     | File                   |
| ----------------------------- | ---------------------- |
| PadoGrid                      | padogrid/padogrid.yaml | 
| Hazelcast                     | hazelcast/values.yaml  | 

## 2. Create OpenShift Project

Let's create the `oc-helm` project. You can create a project with a different name but make sure replace `oc-helm` with your project name throughout this article.

```bash
oc new-project oc-helm
```

## 3. CRC Users (Optional): Create Mountable Persistent Volumes in Master Node

**This section is optional and only applies to CRC users.**

If you are logged onto CRC running on your local PC instead of OpenShift Container Platform (OCP), then to allow read/write permissions, we need to create additional persistent volumes using **hostPath** for PadoGrid. PadoGrid stores workspaces in the `/opt/padogrid/workspaces` directory, which can be optionally mounted to a persistent volume. Let’s create a couple of volumes in the master node as follows.

```bash
# Login to the master node
ssh -i ~/.crc/machines/crc/id_rsa core@$(crc ip)

# Create hostPath volumes. We only need one but let's create two (2)
# in case you want to run addional pods.
sudo mkdir -p /mnt/vol1
sudo mkdir -p /mnt/vol2
sudo chmod -R 777 /mnt/vol1
sudo chmod -R 777 /mnt/vol2
sudo chcon -R -t svirt_sandbox_file_t /mnt/vol1 /mnt/vol2
sudo restorecon -R /mnt/vol1 /mnt/vol2
exit
```

We will use the volumes created as follows:

| Container     | CDC File               | Container Path           | Volume Path |
| ------------- | ---------------------- | ------------------------ | ----------- |
| PadoGrid      | padogrid/padogrid.yaml | /opt/padogrid/workspaces | /mnt/vol?   |

We can now create the required persistent volumes using **hostPath** by executing the following.

```bash
cd_k8s oc_helm; cd padogrid
oc create -f pv-hostPath.yaml
```

## 4. Add User to `anyuid` SCC (Security Context Constraints)

PadoGrid runs as a non-root user that requires read/write permissions to the persistent volume. Let's add your project's **default** user to the **anyuid** SCC.

```bash
oc edit scc anyuid
```

**anyuid SCC:**

Add your project under the users: section. For example, if your project is `oc-helm` then add the following line.

```yaml
users:
- system:serviceaccount:oc-helm:default
```

## 5. Launch Hazelcast

By default, the `start_hazelcast` script launches Hazelcast Enterprise. To run, OSS, specify the `-oss` as shown in the sequent section.

### 5.1. Hazelcast OSS

```bash
cd_k8s oc_helm; cd bin_sh
./start_hazelcast -oss
```

Hazelcast has been configured with `securityContext` enabled. It might fail to start due to the security constraint set by `fsGroup`. Check the StatefulSet events using the describe command as follows.

```bash
oc describe statefulset oc-helm-hazelcast
```

Output:

```console
...
Events:
  Type     Reason        Age                 From                    Message
  ----     ------        ----                ----                    -------
  Warning  FailedCreate  22s (x14 over 63s)  statefulset-controller  create Pod oc-helm-hazelcast-0 in StatefulSet oc-helm-hazelcast failed error: pods "oc-helm-hazelcast-0" is forbidden: unable to validate against any security context constraint: [fsGroup: Invalid value: []int64{1000690000}: 1000690000 is not an allowed group spec.containers[0].securityContext.securityContext.runAsUser: Invalid value: 1000690000: must be in the ranges: [1000570000, 1000579999]]
```

If you see the warning event similar to the above then you need to enter the valid value in the `hazelcast/values.yaml` file as follows.

```bash
cd_k8s oc_helm_wan
vi hazelcast/values.yaml
```

For our example, we would enter a valid value in the `values.yaml` file as follows.

```yaml
# Security Context properties
securityContext:
  # enabled is a flag to enable Security Context
  enabled: true
  # runAsUser is the user ID used to run the container
  runAsUser: 1000570000
  # runAsGroup is the primary group ID used to run all processes within any container of the pod
  runAsGroup: 1000570000
  # fsGroup is the group ID associated with the container
  fsGroup: 1000570000
...
```

After making the changes, restart (stop and start) the Hazelcast cluster as follow.

```bash
cd_k8s oc_helm_wan; cd bin_sh
./stop_hazelcast
./start hazelcast
```

View the Hazelcast services.

```bash
oc get svc
```

Output:

```console
NAME                          TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                        AGE
oc-helm-hazelcast             ClusterIP      None            <none>        5701/TCP                       8s
oc-helm-hazelcast-mancenter   LoadBalancer   172.30.239.38   <pending>     8080:30974/TCP,443:30853/TCP   8s
```

Run `oc expose svc` to expose services.

```bash
oc expose oc-helm-hazelcast-mancenter
```

Run `oc get route` to get the Management Center URL.

```bash
oc get route
```

Output:

```console
NAME                          HOST/PORT                                                                           PATH   SERVICES                         PORT   TERMINATION   WILDCARD
oc-helm-hazelcast-mancenter   oc-helm-hazelcast-mancenter-oc-helm.apps.7919-681139.cor00005-2.cna.ukcloud.com          oc-helm-hazelcast-mancenter   http                 None
```

Management Center URL: http://oc-helm-hazelcast-mancenter-oc-helm.apps.7919-681139.cor00005-2.cna.ukcloud.com

### 5.2. Hazelcast Enterprise

Launch Hazelcast Enterprise Operator and Hazelcast.

```bash
cd_k8s oc_helm; cd bin_sh
./start_hazelcast
```

Hazelcast has been configured with `securityContext` enabled. It might fail to start due to the security constraint set by `fsGroup`. Check the StatefulSet events using the describe command as follows.

```bash
oc describe statefulset oc-helm-hazelcast
```

Output:

```console
...
Events:
  Type     Reason        Age                 From                    Message
  ----     ------        ----                ----                    -------
  Warning  FailedCreate  22s (x14 over 63s)  statefulset-controller  create Pod oc-helm-hazelcast-enterprise-0 in StatefulSet oc-helm-hazelcast-enterprise failed error: pods "oc-helm-hazelcast-enterprise-0" is forbidden: unable to validate against any security context constraint: [fsGroup: Invalid value: []int64{1000690000}: 1000690000 is not an allowed group spec.containers[0].securityContext.securityContext.runAsUser: Invalid value: 1000690000: must be in the ranges: [1000570000, 1000579999]]
```

If you see the warning event similar to the above then you need to enter the valid value in the `hazelcast/values.yaml` file as follows.

```bash
cd_k8s oc_helm_wan
vi hazelcast/values.yaml
```

For our example, we would enter a valid value in the `values.yaml` file as follows.

```yaml
# Security Context properties
securityContext:
  # enabled is a flag to enable Security Context
  enabled: true
  # runAsUser is the user ID used to run the container
  runAsUser: 1000570000
  # runAsGroup is the primary group ID used to run all processes within any container of the pod
  runAsGroup: 1000570000
  # fsGroup is the group ID associated with the container
  fsGroup: 1000570000
...
```

After making the changes, restart (stop and start) the Hazelcast cluster as follow.

```bash
cd_k8s oc_helm_wan; cd bin_sh
./stop_hazelcast
./start hazelcast
```

View the Hazelcast services.

```bash
oc get svc
```

Output:

```console
NAME                                     TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                        AGE
oc-helm-hazelcast-enterprise             ClusterIP      None            <none>        5701/TCP                       7m8s
oc-helm-hazelcast-enterprise-mancenter   LoadBalancer   172.30.178.54   <pending>     8080:30179/TCP,443:32291/TCP   7m8s
```

Run `oc expose svc` to expose the Management Center service.

```bash
oc expose svc oc-helm-hazelcast-enterprise-mancenter
```

Run `oc get route` to get the Management Center URL.

```bash
oc get route
```

Output:

```console
NAME                                     HOST/PORT                                                                                      PATH   SERVICES                                    PORT   TERMINATION   WILDCARD
oc-helm-hazelcast-enterprise-mancenter   oc-helm-hazelcast-enterprise-mancenter-oc-helm.apps.7919-681139.cor00005-2.cna.ukcloud.com          oc-helm-hazelcast-enterprise-mancenter   http                 None
```

Management Center URL: http://oc-helm-hazelcast-enterprise-mancenter-oc-helm.apps.7919-681139.cor00005-2.cna.ukcloud.com

## 6. Launch PadoGrid

### 6.1. CRC Users

```bash
cd_k8s oc_helm; cd bin_sh

# If you have not created local-storage
./start_padogrid

# If you have created local-storage (see Section 3)
./start_padogrid local-storage
```

### 6.2. OCP Users

```bash
cd_k8s oc_helm; cd bin_sh
./start_padogrid
```

## 7. Ingest Data

Login to the PadoGrid pod.

```bash
cd_k8s oc_helm; cd bin_sh
./login_padogrid_pod
```

The `start_padogrid` script automatcially sets the Hazelcast service and the namespace for constructing the DNS address needed by the `perf_test` app to connect to the Hazelcast cluster. This allows us to simply login to the PadoGrid pod and run the `perf_test` app.

*If `perf_test` fails to connect to the Hazelcst cluster then you may need to manually configure the Hazelcast client as described in the [next section](#8-manually-configuring-perf_test).*

Create and run the `perf_test` app.

```bash
create_app
cd_app perf_test; cd bin_sh

# Ingest blob data into Hazelcast.
./test_ingestion -run
```

Read ingested data.

```bash
cd_app perf_test; cd bin_sh
./read_cache eligibility
./read_cache profile
```

The `elibility` and `profile` maps contain blobs. They are meant for carrying out performance tests with different payload sizes. If you want to ingest non-blobs, then you can ingest the Northwind (nw) data. To do so, you must first build the `perf_test` app and run the `test_group` script as shown below.

```bash
cd_app perf_test; cd bin_sh
./build_app

# After the build, run test_group
./test_group -run -prop ../etc/group-factory.properties
```

Read the **nw** data:

```bash
./read_cache nw/customers
./read_cache nw/orders
```

Exit from the PadoGrid pod.

```bash
exit
```

## 8. Manually Configuring `perf_test`

The `test_ingestion` script may fail to connect to the Hazelcast cluster if you started the PadoGrid pod before the Hazelcast cluster is started. In that case, you can simply restart PadoGrid. If it still fails even after the Hazelcast cluster has been started first, then you can manually enter the DNS address in the `etc/hazelcast-client-k8s.xml` file as described below.

```bash
cd_app perf_test
vi etc/hazelcast-client-k8s.xml
```

### 8.1. Hazelcast OSS

Enter the following in the `etc/hazelcast-client-k8s.xml` file. `oc-helm-hazelcast` is the service and  `oc-helm` is the project name.

```xml
                <kubernetes enabled="true">
                        <service-dns>oc-helm-hazelcast.oc-helm.svc.cluster.local</service-dns>
                </kubernetes>
```

### 8.2. Hazelcast Enterprise

Enter the following in the `etc/hazelcast-client-k8s.xml` file. `oc-helm-hazelcast-enterprise` is the service and  `oc-helm` is the project name.

```xml
                <kubernetes enabled="true">
                        <service-dns>oc-helm-hazelcast-enterprise.oc-helm.svc.cluster.local</service-dns>
                </kubernetes>
```

## 9. Teardown

### 9.1. Hazelcast OSS

```bash
cd_k8s oc_helm; cd bin_sh
./cleanup -all -oss
```

### 9.2. Hazelcast Enterprise

```bash
cd_k8s oc_helm; cd bin_sh
./cleanup -all
```

## 10. References

1. Hazelcast Charts, [https://github.com/hazelcast/charts](https://github.com/hazelcast/charts)
2. Configuring Prometheus Metrics, [README-PROM.md](README-PROM.md).
