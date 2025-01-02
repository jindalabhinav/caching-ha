# Caching-HA

This project uses Redis Sentinel for a Highly Available Caching cluster.

## Setup Helm on Windows or Mac

### Install Helm

#### Windows

1. Download the Helm installer from the [Helm Releases](https://github.com/helm/helm/releases).
2. Run the installer and follow the instructions.

#### Mac

Install Helm using Homebrew:

```bash
brew install helm
```

### Add Bitnami Repository

Add the Bitnami repository to Helm:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

### Install Redis Chart

Download the [values.yaml](./values.yaml) file and install the Redis chart with the custom values:

```bash
helm install my-redis bitnami/redis -f values.yaml
```

## Master-Replicas with Sentinel

When installing the chart with `architecture=replication` and `sentinel.enabled=true`, it will deploy a Redis® master StatefulSet (only one master allowed) and a Redis® replicas StatefulSet. In this case, the pods will contain an extra container with Redis® Sentinel. This container will form a cluster of Redis® Sentinel nodes, which will promote a new master in case the actual one fails.

For more details, refer to the [Bitnami Redis Helm Chart](https://github.com/bitnami/charts/tree/main/bitnami/redis).

## Features

- Automatic failover using Redis Sentinel
- Scalable replica configuration
- Persistent storage (2Gi per node)
- Resource-optimized configuration
- Kubernetes-ready deployment
- Comprehensive monitoring

## Prerequisites

- Kubernetes cluster
- Helm v3.x
- Access to bitnami/redis chart
- kubectl CLI tool

## Components

### Redis Cluster
- Master node with defined resource limits
- Three replica nodes for redundancy
- Persistent storage enabled
- Asynchronous replication

### Sentinel
- 3 Sentinel instances
- Quorum of 2 for failover decisions
- Password protection enabled
- Automatic master election

### Test Clients
- [ubuntu-client](./ubuntu-client.yaml) for general testing

## Important Configuration Files

### [values.yaml](./values.yaml)

This file contains the configuration for the Redis Helm chart. Key sections include:

- **global**: Sets global configurations like the default storage class.
- **commonAnnotations**: Adds annotations to all resources.
- **commonLabels**: Adds labels to all resources.
- **master**: Configures the master Redis node, including resource limits and persistence.
- **replica**: Configures the replica Redis nodes, including resource limits and persistence.
- **sentinel**: Enables and configures Redis Sentinel.
- **tls** and auth: Configures TLS and authentication (commented out for production use).
- **metrics**: Enables or disables metrics.

### [ubuntu-client.yaml](./ubuntu-client.yaml)

This file defines a Pod that acts as a client to test the Redis cluster. It includes:

- **metadata**: Contains labels and annotations required for the CIP cluster.
- **containers**: Defines the main container with resource limits.

## Testing High Availability

Once the Redis release is created, follow these steps to test its high availability:

1. **Create an Ubuntu Pod**

    Create an Ubuntu pod to run the necessary commands. Apply the ubuntu-client.yaml file:

    ```bash
    kubectl apply -f ubuntu-client.yaml
    ```

2. **Install Redis Tools and Vim**

    Exec into the Ubuntu pod and install the necessary tools:

    ```bash
    kubectl exec --tty -i ubuntu-client --namespace submission-service -- bash
    apt-get update && apt-get install redis-tools vim -y
    ```



3. **Create and Run the Increment Script**

    Create a script named `increment_script.sh` with the following content:

    ```bash
    #!/bin/bash

    SENTINEL_HOSTS=("sentinel-host-1" "sentinel-host-2" "sentinel-host-3") # replace with actual sentinel host internal DNS names (e.g., squad-redis-node-0.squad-redis-headless.submission-service.svc.cluster.local)
    SENTINEL_PORT=26379
    KEY_NAME="test-key"
    
    while true; do
        MASTER_INFO=""
        for SENTINEL_HOST in "${SENTINEL_HOSTS[@]}"; do
            MASTER_INFO=$(redis-cli -h $SENTINEL_HOST -p $SENTINEL_PORT SENTINEL get-master-addr-by-name mymaster 2>/dev/null)
            if [ -n "$MASTER_INFO" ]; then
                break
            else
                echo "Unable to read from sentinel $SENTINEL_HOST"
            fi
        done
    
        if [ -z "$MASTER_INFO" ]; then
            echo "Failed to get master info from all sentinel hosts. Retrying in 1 second..."
            sleep 1
            continue
        fi
    
        MASTER_IP=$(echo $MASTER_INFO | awk '{print $1}')
        MASTER_PORT=$(echo $MASTER_INFO | awk '{print $2}')
    
        echo "Current master: $MASTER_IP:$MASTER_PORT"
    
        # Attempt to increment the key
        VALUE=$(redis-cli -h $MASTER_IP -p $MASTER_PORT INCR $KEY_NAME 2>/dev/null)
        if [ "$?" -eq 0 ]; then
            echo "Increment successful! Current value: $VALUE (written to $MASTER_IP:$MASTER_PORT)"
        else
            echo "Write failed! Retrying in 1 second... (attempted to write to $MASTER_IP:$MASTER_PORT)"
        fi
    
        sleep 1
        echo # Add a line break
    done
    ```

    Make the script executable and run it:

    **Explanation of the Increment Script**

    The script continuously queries the Redis Sentinel for the current master node and attempts to increment a key (test-key). It handles failovers by iterating through a list of sentinel hosts and retrying if necessary. The logs will show the current master and the result of the increment operation.

4. **Monitor and Interact with the Redis Cluster**

    Open another terminal to monitor and interact with the Redis cluster:

    ```bash
    kubectl get pods --namespace submission-service
    kubectl top pods --containers  --namespace submission-service
    kubectl delete pod <redis-pod-name> --namespace submission-service
    ```

    Replace `<redis-pod-name>` with the name of one of the Redis pods (e.g., `squad-redis-node-0`).

## Observing Failover

- **Get Pods**: Check the status of all pods.
- **Delete Pod**: Simulate a failover by deleting a master or replica pod.
- **Check Sentinel**: Verify that the Sentinel container exists in the same pod as the Redis container.

By following these steps, you can test the high availability of your Redis cluster and observe how it handles failovers.