# Etcd Helm Chart

## Requirements
* Kubernetes >=1.5 (for `StatefulSets` support)
* PV support on the underlying infrastructure
* knowledge about Etcd, see https://etcd.io/docs/v3.4.0/

## StatefulSet Details
* https://kubernetes.io/docs/concepts/abstractions/controllers/statefulsets/

## Chart Details
This chart will do the following:

* Implemented a dynamically scalable etcd cluster using Kubernetes StatefulSets

## Installing the Chart

To install the chart with the release name `my-release`:

```bash
$ helm repo add incubator http://storage.googleapis.com/kubernetes-charts-incubator
$ helm install my-release [-f path/to/my-values.yaml] incubator/etcd
```

## Configuration

The following table lists the configurable parameters of the etcd chart and their default values.

| Parameter                           | Description                          | Default                                            |
| ----------------------------------- | ------------------------------------ | -------------------------------------------------- |
| `image.repository`                  | Container image repository           | `quay.io/coreos/etcd`                              |
| `image.tag`                         | Container image tag                  | `3.4.10`                                           |
| `image.pullPolicy`                  | k8s Image pull policy                | `IfNotPresent`                                     |
| `replicas`                          | k8s StatefulSet replicas. Must not be less than 3. A etcd cluster works only if there are at least 3 nodes. This is then the initial cluster of voters. | `3` |
| `resources`                         | container required resources         | `{}`                                               |
| `clientPort`                        | k8s service port                     | `2379`                                             |
| `peerPorts`                         | Container listening port             | `2380`                                             |
| `storage`                           | Persistent volume size               | `1Gi`                                              |
| `storageClass`                      | Persistent volume storage class      | ``                                                 |
| `annotations`                       | custom annotations for pod           | `{}`                                               |
| `affinity`                          | affinity settings for pod assignment | `{}`                                               |
| `nodeSelector`                      | Node labels for pod assignment       | `{}`                                               |
| `tolerations`                       | Toleration labels for pod assignment | `[]`                                               |
| `extraEnv`                          | Optional environment variables       | `[]`                                               |
| `memoryMode`                        | Using memory as backend storage      | `false`                                            |
| `auth.client.enableAuthentication`  | Enables host authentication using TLS certificates. Existing secret is required.    | `false` |
| `auth.client.secureTransport`       | Enables encryption of client communication using TLS certificates | `false` |
| `auth.peer.useAutoTLS`              | Automatically create the TLS certificates | `false` |
| `auth.peer.secureTransport`         | Enables encryption peer communication using TLS certificates **(At the moment works only with Auto TLS)** | `false` |
| `auth.peer.enableAuthentication`    | Enables host authentication using TLS certificates. Existing secret required | `false` |
| `keepExistingData`                  | Keeps data on disk on node shutdown if persistentVolume is enabled. useful if after a node restart you want rather to continue with existing data from disk than a full sync from cluster. | `false` |
| `logger` |  Specify ‘zap’ for structured logging or ‘capnslog’. WARNING: --logger=capnslog to be deprecated in v3.5. | `zap` |
| `logLevel` | Configures log level. Only supports debug, info, warn, error, panic, or fatal. | `info` |
| `debugMode` | Drop the default log level to DEBUG for all subpackages. default: false (INFO for all packages) | `false` |
| `metrics` | Set level of detail for exported metrics, specify ‘extensive’ to include server side grpc histogram metrics. | `basic` |
| `service.annotations` | Custom annotations for service | `{}` |

* Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`.

* Alternatively, a YAML file that specifies the values for the parameters can be provided while installing the chart. For example,

```bash
$ helm install my-release -f my-values.yaml incubator/etcd
```
> **Tip**: You can use the default [values.yaml](values.yaml)
# To install the chart with secure transport enabled
First you must create a secret which would contain the client certificates: cert, key and the CA which was to used to sign them.
Create the secret using this command:
```bash
$ kubectl create secret generic etcd-client-certs --from-file=ca.crt=path/to/ca.crt --from-file=cert.pem=path/to/cert.pem --from-file=key.pem=path/to/key.pem
```
Deploy the chart with the following flags enabled:
```bash
$ helm install my-release [-f path/to/my-values.yaml] --set auth.client.secureTransport=true --set auth.client.enableAuthentication=true --set auth.client.existingSecret=etcd-client-certs --set auth.peer.useAutoTLS=true incubator/etcd
```
Reference to how to generate the needed certificate:
> Ref: https://coreos.com/os/docs/latest/generate-self-signed-certificates.html

# Deep dive
* https://etcd.io/docs/v3.4.0/op-guide/configuration/
* https://etcd.io/docs/v3.4.0/dev-guide/interacting_v3/

## Cluster Health
* attach to one of the etcd pods using `kubectl exec -it [release-name]-0 [ba]sh`
* in our example the release-name is `etcd-cache`. We expect at least 3 members in the member list:
```bash
$ etcdctl member list
1514aea394120654, started, etcd-cache-1, http://etcd-cache-1.etcd-cache:2380, http://etcd-cache-1.etcd-cache:2379, false
ada8dea6a8b4a498, started, etcd-cache-0, http://etcd-cache-0.etcd-cache:2380, http://etcd-cache-0.etcd-cache:2379, false
aea07e2459ae048e, started, etcd-cache-2, http://etcd-cache-2.etcd-cache:2380, http://etcd-cache-2.etcd-cache:2379, false
```
* check status and health of the current member:
```bash
$ etcdctl endpoint status
127.0.0.1:2379, ada8dea6a8b4a498, 3.4.10, 20 kB, true, false, 2, 14, 14,

$ etcdctl endpoint health
127.0.0.1:2379 is healthy: successfully committed proposal: took = 2.452124ms
```
* to see the status, helth and sync position of all members in the list use the `k8stool` command that provide needed environment:
```bash
$ k8stools etcdctl endpoint status
[INFO] Executing system call etcdctl
http://etcd-cache-0.etcd-cache:2379, ada8dea6a8b4a498, 3.4.10, 20 kB, true, false, 2, 14, 14,
http://etcd-cache-1.etcd-cache:2379, 1514aea394120654, 3.4.10, 20 kB, false, false, 2, 14, 14,
http://etcd-cache-2.etcd-cache:2379, aea07e2459ae048e, 3.4.10, 20 kB, false, false, 2, 14, 14,

$ k8stools etcdctl endpoint health
[INFO] Executing system call etcdctl
http://etcd-cache-0.etcd-cache:2379 is healthy: successfully committed proposal: took = 3.411399ms
http://etcd-cache-1.etcd-cache:2379 is healthy: successfully committed proposal: took = 5.358443ms
http://etcd-cache-2.etcd-cache:2379 is healthy: successfully committed proposal: took = 5.848868ms

$ k8stools etcdctl endpoint hashkv
[INFO] Executing system call etcdctl
http://etcd-cache-0.etcd-cache:2379, 1084519789
http://etcd-cache-1.etcd-cache:2379, 1084519789
http://etcd-cache-2.etcd-cache:2379, 1084519789
```

## Failover
* A failed node is going to be re-joined to a cluster in most situations.

## Scaling using kubectl
* `kubectl scale statefulset [name-of-the-set] --replicas=X`

## Metrics and health
* Metrics are available on the clientPort (default 2379) via endpoint `/metrics`
* for details see https://etcd.io/docs/v3.4.0/metrics/
* health status is available on the clientPort (default 2379) via endpoint `/health`
