# k8s-cbt

This repo consists working prototype for Changed Block Tracking capability in Kubernetes as per the KEP: https://github.com/kubernetes/enhancements/pull/3367


The detailed design is being discussed here: https://docs.google.com/presentation/d/1iIRq2dzSuXktrEm9AjPwyrQIl_EoeqD-BJ7tujoK9C8/edit?usp=sharing


## Diagram

![](./design/cbt-kep.svg)

## Components

1. csi-snapshot-session-manager
    
    - Provides a validating webhook for the CSISnapshotSessionAccess CR
    - Provides the CSISnapshotSessionAccess CR controller

2. external-snapshot-session-service
    
    - Sidecar to the vendor provided SP snapshot session service
    - Validates, translates args and proxies RPC to SP service over unix socket 


3. sample-csi-cbt-service

    - Sample vendor provided SP snapshot service which implements CSI GetDelta rpc to compute Change Block Data
    - Listens requests over unix socket


## Deployment


### csi-snapshot-session-manager

1. Create CRDs

    ```bash
    $ kubectl create -f deploy/crds/
    ```
    
2. Create namespace `csi-snapshot-session-manager`
    
    ```bash
    $ kubectl create namespace csi-snapshot-session-manager
    ```

3. Provision certificate for ValidatingWebhookConfiguration using cert-manager

    ```bash
    $ kubectl create -f deploy/csi-snapshot-session-manager/cert.yaml -n csi-snapshot-session-manager
    ```
    
4. Deploy `csi-snapshot-session-manager` service

    ```bash
    $ kubectl create -f deploy/csi-snapshot-session-manager/csi-snapshot-session-manager.yaml.yaml -n csi-snapshot-session-manager
    ```

### Host path CSI driver

In this prototype, we are using [csi-driver-host-path](https://github.com/kubernetes-csi/csi-driver-host-path) driver as a sample driver. The deploy script has been cloned and refactored to change the default namespace in which csi driver gets deployed

1. Create namespace

    ```bash
    $ kubectl create namespace csi-driver
    ```

2. Deploy hostpath csi driver 
   
    ```bash
    $ ./deploy/deploy.sh
    ```

3. Create hostpath `StorageClass` and `VolumeSnapshotClass`

    ```bash
    $ kubectl create -f deploy/test/csi-storageclass.yaml
    ```

4. (optional) Create test pvc and snapshots

    ```bash
    $ kubectl create -f deploy/test/csi-pvc-block.yaml
    $ kubectl create -f deploy/test/csi-snapshot.yaml
    ```

### external-snapshot-session-service and sample-csi-cbt-service

external-snapshot-session-service and sample-csi-cbt-service are deployed as a sidecars to each other in the single pod

1. Provision TLS Certs

    ```bash
    $ cd deploy/external-snapshot-session-service/cert
    $ ./gen
    ```

2. Add TLS certs to deployment manifests

    ```bash
    $ kubectl create secret tls ext-snap-session-svc-certs --namespace=csi-driver --cert=deploy/external-snapshot-session-service/cert/server-cert.pem --key=deploy/external-snapshot-session-service/cert/server-key.pem 
    ```

3. Deploy external-snapshot-session-service

    ```bash
    $ kubectl create -f deploy/external-snapshot-session-service/ext-snap-session-svc.yaml -n csi-driver
    ```

4. Create `CSISnapshotSessionServer` resource

    Enacode CA Cert
    
    ```bash
    $ base64 -i ./deploy/external-snapshot-session-service/cert/ca-cert.pem
    ```

    Copy the output and replace `GENERATED_CA_CERT` from `deploy/external-snapshot-session-service/csisnapshotsessionservice.yaml`

    Create `CSISnapshotSessionServer` resources in `csi-driver` namespace

    ```bash
    $ kubectl create -f deploy/external-snapshot-session-service/csisnapshotsessionservice.yaml
    ```

### Test with sample client

1. Deploy client RBAC and Pod

    ```bash
    $ kubectl create namespace cbt-client
    $ kubectl create -f deploy/external-snapshot-session-service/client/
    ```

2. Run grpc-client

    ```bash

    # Find the cbt-client pod name and replace CBT_CLIENT_POD_NAME in the following command with actual pod name

    $ kubectl get pods -n cbt-client
    NAME                          READY   STATUS    RESTARTS   AGE
    cbt-client-84bb4b669c-7f945   1/1     Running   0          8m53s

    $ kubectl exec -n cbt-client CBT_CLIENT_POD_NAME -- /grpc-client -h                                                                     
    Usage of /grpc-client:
    -base string
            base volume snapshot name
    -kubeconfig string
            Paths to a kubeconfig. Only required if out-of-cluster.
    -namespace string
            app namespace (default "default")
    -target string
            target volume snapshot name
    ```

    Sample output

    ```bash
    $ kubectl exec -n cbt-client cbt-client-84bb4b669c-7f945 -- /grpc-client -base snapshot-csi-pvc-jr7tp -target snapshot-csi-pvc-jrx5c -namespace default
    ## Creating CSI Snapshot Session in default namespace...

    Session creation state:  
    2023/05/30 13:54:42 finished
    Session creation state:  Ready
    Session params:  {
    "sessionState": "Ready",
    "caCert": "xxxxxx",
    "sessionToken": "emZ4Ymc0bWJudnZ0cW5rd2NuY3NqbWtxemZ0Mjd0dnE=",
    "sessionURL": "external-snapshot-session-service.csi-driver:80",
    "expiryTime": "2023-05-30T14:04:41Z"
    }

    ## Making gRPC Call on external-snapshot-session-service.csi-driver:80 endpoint to Get Changed Blocks Metadata...

    Resp received:
    {"volume_size_bytes":1073741824,"block_metadata":[{"byte_offset":1,"size_bytes":1048576},{"byte_offset":2,"size_bytes":1048576}]}
    {"volume_size_bytes":1073741824,"block_metadata":[{"byte_offset":3,"size_bytes":1048576},{"byte_offset":4,"size_bytes":1048576}]}
    {"volume_size_bytes":1073741824,"block_metadata":[{"byte_offset":5,"size_bytes":1048576},{"byte_offset":6,"size_bytes":1048576}]}
    {"volume_size_bytes":1073741824,"block_metadata":[{"byte_offset":7,"size_bytes":1048576},{"byte_offset":8,"size_bytes":1048576}]}
    {"volume_size_bytes":1073741824,"block_metadata":[{"byte_offset":9,"size_bytes":1048576},{"byte_offset":10,"size_bytes":1048576}]}
    {"volume_size_bytes":1073741824,"block_metadata":[{"byte_offset":11,"size_bytes":1048576},{"byte_offset":12,"size_bytes":1048576}]}
    {"volume_size_bytes":1073741824,"block_metadata":[{"byte_offset":13,"size_bytes":1048576},{"byte_offset":14,"size_bytes":1048576}]}
    {"volume_size_bytes":1073741824,"block_metadata":[{"byte_offset":15,"size_bytes":1048576},{"byte_offset":16,"size_bytes":1048576}]}
    {"volume_size_bytes":1073741824,"block_metadata":[{"byte_offset":17,"size_bytes":1048576},{"byte_offset":18,"size_bytes":1048576}]}
    {"volume_size_bytes":1073741824,"block_metadata":[{"byte_offset":19,"size_bytes":1048576},{"byte_offset":20,"size_bytes":1048576}]}
    ```
