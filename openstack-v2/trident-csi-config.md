# Trident CSI Installation

1. **Deploy Trident operator using Helm on the on-prem cluster**
   ```
   wget https://github.com/NetApp/trident/releases/download/v24.10.0/trident-installer-24.10.0.tar.gz
   tar -xf trident-installer-24.10.0.tar.gz
   cd trident-installer
   helm install trident trident-operator-100.2410.0.tgz -–namespace trident --create-namespace
   ```
   *Trident Operator Running:*
   ```
   # kubectl get pods -n trident
   NAME                 READY  STATUS  RESTARTS    AGE
   trident-controller-598cbb96df-6jlj2  6/6   Running  0        15m
   trident-node-linux-79jpt       2/2   Running  284 (13m ago)  26h
   trident-node-linux-kpnqv       2/2   Running  0        26h
   trident-node-linux-sch5x       2/2   Running  6 (15m ago)   21m
   ```
2. **Configure Trident Backend**

   a. Create a `secret` for SVM admin user and password in `trident` namespace
    ```
    apiVersion: v1
    kind: Secret
    metadata:
      name: backend-netapp-san-secret
    type: Opaque
    stringData:
      username: vsadmin  # SVM Admin user
      password: <SVM admin password>
    ```
   b. Create tridentBackendConfig in `trident` namespace
    ```
    apiVersion: trident.netapp.io/v1
    kind: TridentBackendConfig
    metadata:
      name: backend-fsx-ontap-san
    spec:
      version: 1
      storageDriverName: ontap-san  # "ontap-san" Driver for iSCSI Protocol ("ontap-nas" for NFS)
      managementLIF: 10.0.15.182  # SVM Management Endpoint IP
      svm: fsx01  # SVM Name
      credentials: 
        name: backend-netapp-san-secret
    ```
   c. Create Storage Class
    ```
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: pcd-sc
    provisioner: csi.trident.netapp.io
    parameters:
      backendType: "ontap-san"
      fsType: "ext4"
    allowVolumeExpansion: True
    reclaimPolicy: Delete
    volumeBindingMode: WaitForFirstConsumer
    ```
