# k8s-cbt

This repo consists working prototype for Changed Block Tracking capability in Kubernetes as per the KEP: https://github.com/kubernetes/enhancements/pull/4082


The detailed design is being discussed here: https://docs.google.com/presentation/d/1rZqXbyPcdCtOuexjke4hFIQRQ2FEpVP7VYdZpdAEhy4/edit?usp=sharing


## Diagram

![](./design/cbt-kep.svg)

## Components

1. external-snapshot-metadata
    
    - Sidecar to the vendor provided SP snapshot session service
    - Authenticate the K8s client with the TokenReview API
    - Authorize the K8s client with the SubjectAccessReview API
    - Translates args and proxies RPC to SP service over unix socket 

2. sample-csi-cbt-service

    - Sample vendor provided SP snapshot service which implements CSI GetDelta rpc to compute Change Block Data
    - Listens requests over unix socket


## Deployment


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
    $ kubectl create namespace database
    $ kubectl create -f deploy/test/csi-pvc-block.yaml -n database
    ```

    Create at least 2 snapshots
    ```bash
    $ kubectl create -f deploy/test/csi-snapshot.yaml -n database
    volumesnapshot.snapshot.storage.k8s.io/snapshot-csi-pvc-729hw created
    $ kubectl create -f deploy/test/csi-snapshot.yaml -n database
    volumesnapshot.snapshot.storage.k8s.io/snapshot-csi-pvc-sxkj5 created
    ```

### external-snapshot-metadata service and sample-csi-cbt-service

external-snapshot-metadata service and sample-csi-cbt-service are deployed as a sidecars to each other in the single pod

1. Provision TLS Certs

    ```bash
    $ cd deploy/external-snapshot-metadata/cert/
    $ ./gen.sh
    ```

2. Add TLS certs to deployment manifests

    Go to project root directory and execute the following commands
    ```bash
    $ kubectl create secret tls ext-snap-metadata-certs --namespace=csi-driver --cert=deploy/external-snapshot-metadata/cert/server-cert.pem --key=deploy/external-snapshot-metadata/cert/server-key.pem 
    ```

3. Create `snapshotmetadataservices.cbt.storage.k8s.io` CRD
    ```bash
    $ kubectl create -f external-snapshot-metadata/deploy/crd
    ```
5. Deploy external-snapshot-metadata service

    ```bash
    $ kubectl create -f deploy/external-snapshot-metadata/ext-snap-metadata-svc.yaml -n csi-driver
    ```

6. Create `SnapshotMetadataService` resource

    Enacode CA Cert
    
    ```bash
    $ base64 -i ./deploy/external-snapshot-metadata/cert/ca-cert.pem
    ```

    Copy the output and replace `GENERATED_CA_CERT` from `deploy/external-snapshot-metadata/snapshotmetadataservice.yaml`

    Create `SnapshotMetadataService` resources in `csi-driver` namespace

    ```bash
    $ kubectl create -f deploy/external-snapshot-metadata/snapshotmetadataservice.yaml
    ```

### Test with sample client

1. Deploy client RBAC and Pod

    ```bash
    $ kubectl create namespace cbt-client
    $ kubectl create -f  deploy/external-snapshot-metadata/client/client.yaml
    ```

2. Run grpc-client

    ```bash

    # Find the cbt-client pod name and replace CBT_CLIENT_POD_NAME in the following command with actual pod name

    $ kubectl get pods -n cbt-client
    NAME                        READY   STATUS    RESTARTS   AGE
    cbt-client-d854dbbf-88rdd   1/1     Running   0          69m

    $ kubectl exec -n cbt-client cbt-client-d854dbbf-88rdd -- /grpc-client -h
    Usage of /grpc-client:
    -base string
            base volume snapshot name
    -client-namespace string
            client namespace (default "default")
    -kubeconfig string
            Paths to a kubeconfig. Only required if out-of-cluster.
    -namespace string
            snapshot namespace (default "default")
    -service-account string
            client service account (default "default")
    -target string
            target volume snapshot name
    ```

    Sample output

    ```bash
    $ kubectl exec -n cbt-client cbt-client-d854dbbf-88rdd -- /grpc-client -base snapshot-csi-pvc-lppqh -target snapshot-csi-pvc-s2pmn -namespace database -client-namespace cbt-client -service-account cbt-client

    ## Discovering SnapshotMetadataService for the driver and creating SA Token 

    2023/08/08 14:02:37 Finding driver name for the snapshots
    2023/08/08 14:02:37 Search SnapshotMetadataService object for driver: hostpath.csi.k8s.io
    2023/08/08 14:02:37 Found SnapshotMetadataService object csi-hostpath-xw2s8 for driver: hostpath.csi.k8s.io
    2023/08/08 14:02:37 Creating SA Token using TokenRequest resource

    ## Making gRPC Call on external-snapshot-metadata.csi-driver:6443 endpoint to Get Changed Blocks Metadata...

    2023/08/08 14:02:37 TokenRequest Response:: {
    "metadata": {
        "name": "cbt-client",
        "namespace": "cbt-client",
        "creationTimestamp": "2023-08-08T14:02:37Z",
    },
    "spec": {
        "audiences": [
        "005e2583-91a3-4850-bd47-4bf32990fd00"
        ],
        "expirationSeconds": 600,
        "boundObjectRef": null
    },
    "status": {
        "token": "XXXXXXXXXX",
        "expirationTimestamp": "2023-08-08T14:12:37Z"
    }
    }
    Resp received:
    {"volume_size_bytes":1073741824,"block_metadata":[{"byte_offset":1,"size_bytes":1048576},{"byte_offset":2,"size_bytes":1048576}]}
    {"volume_size_bytes":1073741824,"block_metadata":[{"byte_offset":3,"size_bytes":1048576},{"byte_offset":4,"size_bytes":1048576}]}
    {"volume_size_bytes":1073741824,"block_metadata":[{"byte_offset":5,"size_bytes":1048576},{"byte_offset":6,"size_bytes":1048576}]}
    {"volume_size_bytes":1073741824,"block_metadata":[{"byte_offset":7,"size_bytes":1048576},{"byte_offset":8,"size_bytes":1048576}]}
    2023/08/08 14:02:37 finished
    {"volume_size_bytes":1073741824,"block_metadata":[{"byte_offset":9,"size_bytes":1048576},{"byte_offset":10,"size_bytes":1048576}]}
    {"volume_size_bytes":1073741824,"block_metadata":[{"byte_offset":11,"size_bytes":1048576},{"byte_offset":12,"size_bytes":1048576}]}
    {"volume_size_bytes":1073741824,"block_metadata":[{"byte_offset":13,"size_bytes":1048576},{"byte_offset":14,"size_bytes":1048576}]}
    {"volume_size_bytes":1073741824,"block_metadata":[{"byte_offset":15,"size_bytes":1048576},{"byte_offset":16,"size_bytes":1048576}]}
    {"volume_size_bytes":1073741824,"block_metadata":[{"byte_offset":17,"size_bytes":1048576},{"byte_offset":18,"size_bytes":1048576}]}
    {"volume_size_bytes":1073741824,"block_metadata":[{"byte_offset":19,"size_bytes":1048576},{"byte_offset":20,"size_bytes":1048576}]}
    ```
