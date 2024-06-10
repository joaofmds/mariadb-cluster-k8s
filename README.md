# MariaDB Galera Cluster on Kubernetes

This project sets up a highly available MariaDB Galera Cluster on Kubernetes. The cluster consists of 3 MariaDB instances, configured for synchronous replication and fault tolerance.

## Table of Contents
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Components](#components)
  - [ConfigMap](#configmap)
  - [StatefulSet](#statefulset)
  - [Services](#services)
- [Deployment](#deployment)
- [Testing](#testing)
- [Cleanup](#cleanup)

## Architecture

1. **StatefulSet**: Manages the deployment and scaling of MariaDB instances, ensuring stable identities.
2. **ConfigMap**: Stores the MariaDB configuration.
3. **Services**: Includes a headless service for internal discovery and a LoadBalancer service for external access.

## Prerequisites

- Kubernetes cluster (minikube, GKE, EKS, etc.)
- kubectl configured to interact with your cluster

## Components

### ConfigMap

Stores the MariaDB configuration required for the Galera Cluster.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mariadb-config
data:
  my.cnf: |
    [mysqld]
    query_cache_size=0
    binlog_format=ROW
    default_storage_engine=InnoDB
    innodb_autoinc_lock_mode=2
    bind-address=0.0.0.0
    [galera]
    wsrep_on=ON
    wsrep_provider=/usr/lib/galera/libgalera_smm.so
    wsrep_cluster_address=gcomm://mariadb-0.mariadb,mariadb-1.mariadb,mariadb-2.mariadb
    wsrep_cluster_name="mariadb_cluster"
    wsrep_node_address=$(POD_IP)
    wsrep_node_name=$(HOSTNAME)
    wsrep_sst_method=rsync
```

### StatefulSet

Manages the MariaDB instances and ensures each instance has a stable network identity and persistent storage.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mariadb
spec:
  serviceName: "mariadb"
  replicas: 3
  selector:
    matchLabels:
      app: mariadb
  template:
    metadata:
      labels:
        app: mariadb
    spec:
      containers:
      - name: mariadb
        image: mariadb:10.5
        ports:
        - containerPort: 3306
          name: mariadb
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "password"
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        volumeMounts:
        - name: config
          mountPath: /etc/mysql/conf.d
        - name: data
          mountPath: /var/lib/mysql
      volumes:
      - name: config
        configMap:
          name: mariadb-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

### Services

#### Headless Service

Enables internal service discovery for the MariaDB instances.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mariadb
spec:
  clusterIP: None
  ports:
  - port: 3306
    name: mariadb
  selector:
    app: mariadb
```

#### LoadBalancer Service

Exposes the MariaDB cluster for external access.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mariadb-lb
spec:
  type: LoadBalancer
  ports:
  - port: 3306
    targetPort: 3306
    protocol: TCP
  selector:
    app: mariadb
```

## Deployment

1. Clone the repository:

    ```bash
    git clone https://github.com/yourusername/mariadb-galera-cluster-k8s.git
    cd mariadb-galera-cluster-k8s
    ```

2. Apply the ConfigMap:

    ```bash
    kubectl apply -f configmap.yaml
    ```

3. Apply the StatefulSet:

    ```bash
    kubectl apply -f statefulset.yaml
    ```

4. Apply the Services:

    ```bash
    kubectl apply -f headless-service.yaml
    kubectl apply -f loadbalancer-service.yaml
    ```

## Testing

1. Obtain the external IP address of the MariaDB service:

    ```bash
    kubectl get services mariadb-lb
    ```

2. Connect to the MariaDB cluster using a MySQL client:

    ```bash
    mysql -h <external-ip> -P 3306 -u root -p
    ```

3. Perform some queries to ensure the database is operational and data is replicated across the nodes.

## Cleanup

To delete the deployment, run:

```bash
kubectl delete -f loadbalancer-service.yaml
kubectl delete -f headless-service.yaml
kubectl delete -f statefulset.yaml
kubectl delete -f configmap.yaml
```
