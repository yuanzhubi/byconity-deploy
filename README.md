# ByConity Deploy

This repos demonstrates how to deploy a ByConity cluster in your Kubernetes cluster.

## Deploy local to try a demo

### Prerequisites

- Install and setup [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) in your local environment
- Install [helm](https://helm.sh/) in your local environment
- Install [kind](https://kind.sigs.k8s.io/) and [Docker](https://www.docker.com/)

### File Introduction
```-- byconity
|-- Chart.yaml                     # YAML file containing information about the byconity chart
|-- charts
|   |-- fdb-operator               # fbd-operator Chart
|   `-- hdfs                       # hdfs Chart
|-- files                          # Byconity components config file template
|   |-- cnch-config.yaml           # config file template for some significant configurations
|   |-- daemon-manager.yaml        # config file template for daemon-manager
|   |-- resource-manager.yaml      # config file template for resource-manager
|   |-- server.yaml                # config file template for cnch-server
|   |-- tso.yaml                   # config file template for tso
|   |-- users.yaml                 # config file template for user configutation
|   `-- worker.yaml                # config file template for byconity-worker
|-- templates                      # A directory of templates that, when combined with values,
                                   # will generate valid Kubernetes manifest files.
`-- values.yaml                    # The default configuration values for this chart
```


### Use Kind to configure a local Kubernetes cluster

> Warning: kind is not designed for production use.
> Note for macOS users: you may need to [increase the memory available](https://docs.docker.com/desktop/get-started/#resources) for containers (recommend 6 GB).

This would create a 1-control-plane, 3-worker Kubernetes cluster.


```bash
kind create cluster --config examples/kind/kind-byconity.yaml
```

Test to ensure the local kind cluster is ready:

```bash
kubectl cluster-info
```

### Initialize the Byconity demo cluster

```bash
# Install with fdb CRD first
helm upgrade --install --create-namespace --namespace byconity -f ./examples/kind/values-kind.yaml byconity ./chart/byconity --set fdb.enabled=false

# Install with fdb cluster
helm upgrade --install --create-namespace --namespace byconity -f ./examples/kind/values-kind.yaml byconity ./chart/byconity
```

Wait until all the pods are ready.

```bash
kubectl -n byconity get po
```

Let's try it out!

```
$ kubectl -n byconity exec -it sts/byconity-server -- bash
root@byconity-server-0:/# clickhouse client

172.16.1.1 :)
```

### Delete or stop ByConity from your Kubernetes cluster

```bash
helm uninstall --namespace byconity byconity
```

In case you want to stop it temporarily stop it on your machine

```bash
docker stop byconity-cluster-control-plane byconity-cluster-worker byconity-cluster-worker2 byconity-cluster-worker3
```

## Deploy in your self-built Kubernetes

### How to deploy or buy a Kubernetes cluster?

You can get information here: [Production environment](https://kubernetes.io/docs/setup/production-environment/)

### Prerequisites

- Install and setup [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) in your local environment
- Install [helm](https://helm.sh/) in your local environment

### Prepare your storage provider

For best [TCO](https://en.wikipedia.org/wiki/Total_cost_of_ownership) with performance, local disks are preferred to be used with ByConity servers and workers.

> Storage for ByConity servers and workers is for disk cache only, you can delete them any time.

You may use storage providers like [OpenEBS local PV](https://openebs.io/docs/concepts/localpv).

### Prepare your own helm values files

You may copy from ./chart/byconity/values.yaml and modify some fields like:
there is an example located in examples/k8s/values.yaml
- storageClassName
  for example
```dash
storage:
    storageClassName: openebs-hostpath
```
- timezone
- replicas for server/worker
- hdfs storage request

  if you want to use your own existing hdfs cluster please set hdfs.enabled=true
  you can override the hdfs address configuration in values.yaml
```dash
byconity:
  hdfs_address: hdfs://your own hdfs:port
hdfs:
  enabled: false
```
- fdb configuration

  if you want to use your own fdb. please set fdb.enabled=false and fdb-operator.enabled=false
  you can refer to values_use_existing_fdb.yaml
```dash
byconity:
  hdfs_address: hdfs://byconity-hdfs-namenodes:8020 # can using your own hdfs
  use_existing_fdb: true
  fdb_cluster_file: your fdb-cluster-file content. #byconity_fdb:Is0hBgl6iICdHuspBmhAODmD5WISXKzI@192.168.224.150:4501,192.168.226.83:4501,192.168.228.152:4501
fdb:
  enabled: false
fdb-operator:
  enabled: false
```

### Initialize the Byconity cluster

```bash
# Install with fdb CRD first
helm upgrade --install --create-namespace --namespace byconity -f ./your/custom/values.yaml byconity ./chart/byconity --set fdb.enabled=false

# Install with fdb cluster
helm upgrade --install --create-namespace --namespace byconity -f ./your/custom/values.yaml byconity ./chart/byconity
```

Wait until all the pods are ready.

```bash
kubectl -n byconity get po
```

Let's try it out!

```
$ kubectl -n byconity exec -it sts/byconity-server -- bash
root@byconity-server-0:/# clickhouse client

172.16.1.1 :)
```

## Test

### Run some SQLs to test

```sql
CREATE DATABASE IF NOT EXISTS test;
USE test;
DROP TABLE IF EXISTS test.lc;
CREATE TABLE test.lc (b LowCardinality(String)) engine=CnchMergeTree ORDER BY b;
INSERT INTO test.lc SELECT '0123456789' FROM numbers(100000000);
SELECT count(), b FROM test.lc group by b;
DROP TABLE IF EXISTS test.lc;
DROP DATABASE test;
```

## Update your Byconity cluster

### Add new virtual warehouses

Assume you want to add 2 virtual warehouses for your users, 5 replicas for `my-new-vw-default` to read and 2 replicas for `my-new-vw-write` to write.

Modify your values.yaml

```yaml
byconity:
  virtualWarehouses:
    ...

    - <<: *defaultWorker
      name: my-new-vw-default
      replicas: 5
    - <<: *defaultWorker
      name: my-new-vw-write
      replicas: 2
```

We can create new Virtual Warehouse while the cluster is running. Please refer to [Byconity ClusterMode](https://byconity.github.io/docs/basic-guide/virtual-warehouse-configuration#cluster-mode)

Apply your helm release with your new values

```bash
helm upgrade --install --create-namespace --namespace byconity -f ./your/custom/values.yaml byconity ./chart/byconity
```

Run `CREATE WAREHOUSE` DDL to create logic virtual warehouse in Byconity

```sql
CREATE WAREHOUSE IF NOT EXISTS `my-new-vw-default` SETTINGS num_workers = 0, type = 'Read';
CREATE WAREHOUSE IF NOT EXISTS `my-new-vw-write` SETTINGS num_workers = 0, type = 'Write';
```

Done. Let's create a table and use your new virtual warehouse now!

```sql
-- Create a table with SETTINGS cnch_vw_default = 'my-new-vw-default', cnch_vw_write = 'my-new-vw-write'
CREATE DATABASE IF NOT EXISTS test;
CREATE TABLE test.lc2 (b LowCardinality(String)) engine=CnchMergeTree
ORDER BY b
SETTINGS cnch_vw_default = 'my-new-vw-default', cnch_vw_write = 'my-new-vw-write';

-- Check if the table has the new settings
SHOW CREATE TABLE test.lc2;
```
