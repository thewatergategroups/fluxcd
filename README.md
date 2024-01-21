## Flux CD
### Installation 
- https://fluxcd.io/flux/installation/
1. `curl -s https://fluxcd.io/install.sh | sudo bash`

### Provisioning 
1. `make provision` 
2. Need to manually add repo creds for the argocd repos 
    - add the below to repo-creds.yaml and replace password and username with your git token 
```yaml
kubectl apply -f <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: private-repo-creds
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repo-creds
stringData:
  type: git
  url: https://github.com/1ndistinct
  password: my-password
  username: my-username
```
    - run to apply `make apply-argo-repo-creds`
3. Port forward argocd to use it `kubectl port-forward svc/argocd-server -n argocd 8080:443`

## Charts
- https://truecharts.org/charts/description_list#Stable

### Add taint and label to node that is running wireguard 
- Due to the network routing, other pods won't have internet connection. We can use this node primarily for storage and pods that don't need internet. Lets add a taint to this node to say as much. 
- `kubectl taint nodes node1 networkmode=host:PreferNoSchedule`
- `kubectl label nodes node1 networkmode=host`
- all pods and deployments that don't need outbound access should be scheduled on this node 

### Add Storage:
- `microk8s enable rook-ceph`
- create a ceph cluster with the below 
```bash
kubectl apply -n rook-ceph -f - <<EOF
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephVersion:
    image: quay.io/ceph/ceph:v17.2.6
  dataDirHostPath: /var/lib/rook
  mon:
    count: 3
    allowMultiplePerNode: true
  dashboard:
    enabled: true
  # cluster level storage configuration and selection
  storage:
    useAllNodes: true
    useAllDevices: true
  placement:
    all:
      tolerations:
        - effect: PreferNoSchedule
          key: networkmode
          operator: Equal
          value: host
EOF
```
##### BLOCK STORAGE
- install storage class for block storage [docs](https://rook.io/docs/rook/v1.12/Storage-Configuration/Block-Storage-RBD/block-storage/#provision-storage)7
  - IF YOU'RE HAVING ISSUES: 
    - install rook ceph toolbox on kuber [plugin](https://rook.io/docs/rook/latest-release/Troubleshooting/kubectl-plugin/)
    - run init on replica pool `kubectl rook-ceph rbd pool init replicapool`
  - NOTE: DOESN'T HAVE READWRITEMANY IN FILESYSTEM MODE - USE CEPHFS INSTEAD
```bash 
kubectl apply -n rook-ceph -f - <<EOF
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 1
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: rook-cephfs
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
    clusterID: rook-ceph
    pool: replicapool
    imageFormat: "2"
    imageFeatures: layering
    csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
    csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
    csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
    csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
    csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
    csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
    csi.storage.k8s.io/fstype: ext4
reclaimPolicy: Delete
allowVolumeExpansion: true
EOF
```
##### FILESYSTEM STORAGE
- install storage class for filesystem storage [docs](https://rook.io/docs/rook/v1.12/Storage-Configuration/Shared-Filesystem-CephFS/filesystem-storage/#create-the-filesystem)
```bash
kubectl apply -n rook-ceph -f - <<EOF
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: filesystem
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 1
  dataPools:
    - name: replicated
      replicated:
        size: 1
  preserveFilesystemOnDelete: true
  metadataServer:
    activeCount: 1
    activeStandby: true
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-cephfs
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  clusterID: rook-ceph
  fsName: filesystem
  pswool: filesystem-replicated
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph

reclaimPolicy: Delete
EOF
```

