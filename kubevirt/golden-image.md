# Creating a new VM from Golden Image

Prerequisites:

- install qemu-utils package for image conversion and virtual size detection

1. Download image

```
wget <image>.qcow2
```

*Note*: The image format needs to be `qcow2` or raw ie. `.img`. If the image is an ISO, then you will first need to convert it to `qcow2`.

```
qemu-img convert -O qcow2 /path/to/IMAGE.iso NEW_IMAGE.qcow2
```

2. Verify image size

```
root@p9-master-r440-2:/home/lab# qemu-img info ECV-9.1.4.2_92345.qcow2
image: ECV-9.1.4.2_92345.qcow2
file format: qcow2
virtual size: 30 GiB (32212254720 bytes)
disk size: 706 MiB
cluster_size: 65536
Format specific information:
    compat: 0.10
    refcount bits: 16
root@p9-master-r440-2:/home/lab#
```

3. Knowing the virtual image size (30GiB) from previous step, create and kubectl apply a new "Golden Image" DataVolume with storage just slightly larger to account for any overhead. Typically, 2Gib larger should be fine.

```
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: ecv9-1-4-2-92345-golden-dv
spec:
  source:
    http:
      url: "http://198.19.99.7/SilverPeak/ECV-9.1.4.2_92345.qcow2"
  pvc:
    accessModes:
      - ReadWriteMany
    resources:
      requests:
        storage: 32Gi
    storageClassName: rook-ceph-block
    volumeMode: Block
```

4. Apply the DataVolume with `kubectl`

```
kubectl apply -f volumes/ecv-golden-dv.yaml
```

```
$ watch kubectl get pv,pvc,dv
NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                 STORAGECLASS      REASON   AGE
persistentvolume/pvc-81ec39c9-c40c-4197-b1ad-4d9189d9195a   32Gi       RWX            Delete           Bound    default/ecv9-1-4-2-92345-golden-dv        rook-ceph-block                  42s

NAME                                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
persistentvolumeclaim/ecv9-1-4-2-92345-golden-dv        Bound    pvc-81ec39c9-c40c-4197-b1ad-4d9189d9195a   32Gi       RWX            rook-ceph-block         43s

NAME                                     PHASE              PROGRESS   RESTARTS   AGE
datavolume.cdi.kubevirt.io/ecv9-1-4-2-92345-golden-dv        ImportInProgress   44.11%                43s
$
```

5. Create `VirtualMachine` manifest, cloning from the original Golden DataVolume created above. Special care should be taken to ensure the `memory:` is a multiple of the `cpu:` value.
See: [Production Example Manifests](https://github.com/platform9/cs-ntt/blob/main/kubevirt/virtualmachines)

```
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: ecv-demo-1
spec:
  runStrategy: Always
  template:
    metadata:
      labels:
        debugLogs: "true"
      # Only required for VMs with DPDK interfaces
      annotations:
        kubevirt.io/memfd: "false"
    spec:
      domain:
        cpu:
          cores: 8
        resources:
          requests:
            # Memory has to be a multiple of the cpu count
            memory: 8Gi
            cpu: 8
        memory:
          guest: 8Gi
          # Only required for VMs with DPDK interfaces
          hugepages:
            pageSize: "1Gi"
        devices:
          interfaces:
            # This is a DPDK interface (optional)
            # Until PMK 5.8 is released, the name of the DPDK interfaces
            # must be in the format: 'vhost-user-net-x'
            - name: vhost-user-net-1
              vhostuser: {}
            # This is a non-DPDK interface (optional)
            # These interfaces names are free form, they do not need to conform to any naming convention.
            - name: net-2
              bridge: {}
            # This is the POD network interface (required)
            # Typically, this will be the last interface in the list so that the VM's management
            # interface will not be set on this interface. This interface is required for Live Migration.
            - name: default
              masquerade: {}
          disks:
            - name: volume-1
              disk:
                bus: virtio
      networks:
        - name: vhost-user-net-1
          multus:
            networkName: net1-dpdk-3302
        - name: net-2
          multus:
            networkName: net1-nondpdk-98
        - name: default
          pod: {}
      volumes:
        - dataVolume:
            name: ecv-demo-1-dv
          name: volume-1
  dataVolumeTemplates:
    - metadata:
        name: ecv-demo-1-dv
      spec:
        pvc:
          accessModes:
            - ReadWriteMany
          resources:
            requests:
              storage: 32Gi
          storageClassName: rook-ceph-block
          volumeMode: Block
        source:
          pvc:
            namespace: default
            name: ecv9-1-4-2-92345-golden-dv
```

6. Apply the manifest with `kubectl`

```
kubectl apply -f virtualmachines/ecv-demo-1.yaml
```

7. Watch the DataVolume clone and the VirtualMachine launch:

```
watch kubectl get pv,pvc,dv,vm,vmi
```
