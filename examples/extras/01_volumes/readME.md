# Volumes In Kubernetes 

## Syllabus : -
1. What Are Volumes in Kubernetes?
2. Why Do We Use Volumes?
3. HostPath Volume
4. PersistentVolume
5. PersistentVolumeClaim
6. How PV and PVC Work Together
7. Dynamic Provisioning

### 1. **What Are Volumes in Kubernetes?**

In Kubernetes, a **volume** is a directory that can store data, which is accessible to the containers in a pod. Volumes ensure that data is preserved across container restarts within a pod, addressing the ephemeral nature of containers. When a container crashes or restarts, the data inside the container's filesystem is lost. However, data stored in a Kubernetes volume persists as long as the pod exists (and in some cases, even after the pod is deleted, depending on the volume type).

### 2. **Why Do We Use Volumes?**
Volumes are used in Kubernetes to address several important use cases:
- **Persistent Data**: Containers are stateless by nature, meaning their local filesystem data is lost when the container is stopped or restarted. Volumes provide a way to store data persistently.
- **Shared Storage**: When multiple containers are running in the same pod, volumes allow them to share data, since they mount the same volume.
- **Separation of Storage from Containers**: Volumes decouple storage from the lifecycle of a container, ensuring that data remains available even when containers are destroyed or restarted.
- **Stateful Applications**: For applications that require state, such as databases, volumes allow you to persist that state across pod or node failures.

### 3. **HostPath Volume**

A **HostPath** volume mounts a file or directory from the host node’s filesystem into a pod. This is often used in environments where containers need to interact with the host system, such as for accessing logs, configuration files, or exposing directories from the host.

- **Why Use HostPath?**
  - **Access to Node-Specific Files**: Useful when the container needs access to specific files or directories on the node’s filesystem (like `/var/log`).
  - **Testing in Local Clusters**: Can be used for local testing when you don't have a cloud-based PersistentVolume setup.

- **Limitations of HostPath**:
  - Not suitable for cloud-based or multi-node environments since it's tied to a specific node.
  - Can lead to issues when pods are rescheduled to a different node, as the data does not move with the pod.
  - It is available in the lifetime of the pod, once the pod is deleted, then it will be deleted

##### Example of HostPath Volume in a Pod:

```yaml
# pod.kub.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
  labels:
    name: mypod
spec:
  containers:
  - name: mypod
    image: busybox
    command: ['sh', '-c', 'mkdir /data; sleep 3600']
    volumeMounts:
     - mountPath: /data #path on the container
       name: hpvolume
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
  volumes:
    - name: hpvolume
      hostPath:
        path: / # path on cluster
```

##### To Run The Example:

```bash
# apply the yaml file
kubectl apply -f pod.kub.yaml

# enter the minikube to access the container
docker exec -it minikube /bin/bash

# search for our pod and get its ID
docker container ls | grep mypod

# enter the pod 
docker exec -it 19905def0c1a sh 

# list all the directoriess, you will see that the data directory is made
ls

# move to the data directory
cd data

# list all the directories, you will see that it includes all the content of the minikube root directory
# which means that we mount the root of minikube into our pod
ls
```

### 4. **PersistentVolume (PV)**

A **PersistentVolume (PV)** in Kubernetes represents a piece of storage in the cluster that has been provisioned by an administrator or dynamically created using storage classes. It abstracts the physical storage resource and makes it available to users through the Kubernetes API.

- **Key Characteristics of PV**:
  - It is independent of the pod lifecycle. Even if the pod is destroyed, the PV remains available.
  - Supports various types of storage backends, such as local storage, NFS, cloud provider-specific storage (like AWS EBS, Google Cloud Persistent Disks), etc.

#### **Access Modes of PersistentVolume (PV)**
Access modes determine how a PV can be mounted by pods. These are:
1. **ReadWriteOnce (RWO)**: The volume can be mounted as read-write by a single node. Most commonly used for cloud block storage (like AWS EBS, GCE Persistent Disks).
2. **ReadOnlyMany (ROX)**: The volume can be mounted as read-only by many nodes simultaneously. Useful for scenarios where data needs to be shared in a read-only manner.
3. **ReadWriteMany (RWX)**: The volume can be mounted as read-write by many nodes simultaneously. Useful for file storage systems that can handle concurrent writes, like NFS.

##### Example of PersistentVolume:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-example
spec:
  capacity:
    storage: 1Gi             # Size of the PV
  accessModes:
    - ReadWriteOnce          # The PV can be mounted as read-write by a single node
  persistentVolumeReclaimPolicy: Retain  # Reclaim policy
  hostPath:
    path: /mnt/data           # Path on the host node for storage
```

### 5. **PersistentVolumeClaim (PVC)**

A **PersistentVolumeClaim (PVC)** is a request for storage by a user. It "claims" a specific amount of storage from a **PersistentVolume (PV)** and binds the PV to a pod. PVCs are how users request and consume PV resources in Kubernetes.

- **Why Use PVCs?**
  - PVCs allow developers to request storage without worrying about the underlying storage details.
  - By abstracting the storage request, PVCs allow Kubernetes to dynamically match user needs with available storage, making it flexible and scalable.

#### **Reclaim Policy**
The **reclaim policy** determines what happens to a PV once its associated PVC is deleted. The two common policies are:
1. **Retain**: The PV and its data are preserved even after the PVC is deleted. The administrator can then manually reclaim or delete the data.
2. **Delete**: The PV and its data are automatically deleted when the PVC is released.
3. **Recycle**: (Deprecated=not used) The volume’s contents are scrubbed, and it becomes available again for new claims.

##### Example of PersistentVolumeClaim:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-example
spec:
  accessModes:
    - ReadWriteOnce            # The same access mode as in the PV
  resources:
    requests:
      storage: 1Gi             # The PVC requests 0.5MB of storage
  selector:                     # Selector to match the PV based on labels
    matchLabels:
      type: fast-storage        # This label must match the PV's label
```

### 6. **How PV and PVC Work Together**
- A **PersistentVolumeClaim** (PVC) is made by a pod to request storage.
- Kubernetes matches the PVC to a **PersistentVolume** (PV) that meets the requirements (size, access mode).
- The PVC is bound to the PV, and the pod can now use the storage.

##### Example Pod Using PVC:
- Try to apply the previous PV and PVC, then apply the following POD

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-using-pvc
spec:
  containers:
    - name: busybox
      image: busybox
      command: ['sh', '-c', 'echo "Hello from PVC" > /data/hello.txt; sleep 3600']
      volumeMounts:
        - name: pvc-storage
          mountPath: /data           # The path inside the container
  volumes:
    - name: pvc-storage
      persistentVolumeClaim:
        claimName: pvc-example       # The PVC this pod will use

```

- To check if the PV is mounted to a PVC, run the following command to list all the PV 

```bash
kubectl get pv
```
- we will see the following output
```bash
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                 STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pv-example                                 1Gi        RWO            Retain           Available                                        <unset>                          4m11s
pvc-f2bce6e0-2204-4019-bf9e-89cf519487cf   1Gi        RWO            Delete           Bound       default/pvc-example   standard       <unset>                          4m9s
```
- by looking at the status of pvc-f2bce6e0-2204-4019-bf9e-89cf519487cf, you will see that it is BOUND and CLAIMED by default/pvc-example.

- then you can list all the PVC using the following command 
```bash
kubectl get pv
```
- we will see the following output

```bash
NAME          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
pvc-example   Bound    pvc-f2bce6e0-2204-4019-bf9e-89cf519487cf   1Gi        RWO            standard       <unset>                 6m31s
```
- it says that the pvc-example STATUS is BOUND, and the VOLUME at which it is bounded is pvc-f2bce6e0-2204-4019-bf9e-89cf519487cf which is the same volume in the PV

- Also you can check the status of the pod using : 
```bash
kubectl describe pod pod-using-pvc
```

- you will see the following AS A PART OF THE output : -
```bash
Volumes:
  pvc-storage:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  pvc-example
    ReadOnly:   false
```

- which means that our pod claimed PVC named pvc-example :)  


### 7. Dynamic provisioning : -
Dynamic provisioning in Kubernetes is the process where storage resources (such as volumes) are automatically created on-demand when a **PersistentVolumeClaim (PVC)** is made by a pod. Instead of manually pre-creating **PersistentVolumes (PVs)**, Kubernetes will dynamically create the PV based on the **StorageClass** definition.

This is particularly useful in cloud environments (such as AWS, GCP, Azure) or storage systems that support dynamic provisioning because the administrator does not have to create volumes ahead of time, simplifying the storage management process.

#### How Dynamic Provisioning Works:
1. **StorageClass**: Defines the parameters needed for dynamic provisioning, such as the provisioner, storage type, and other storage-specific settings.
2. **PVC**: When a user creates a PVC, they specify the required storage size, access mode, and the `storageClassName` (or use the default one).
3. **Provisioner**: Kubernetes automatically communicates with the provisioner specified in the `StorageClass` (e.g., AWS EBS, Google PD, NFS), and the provisioner creates a PersistentVolume dynamically.
4. **PV Creation**: Once the PVC is created, Kubernetes dynamically provisions a new PV that satisfies the claim's requirements and binds it to the PVC.
5. **Pod Usage**: The pod uses the dynamically created and bound PV for storage.

#### Dynamic Provisioning Example

##### 1. **StorageClass Definition**
This defines how dynamic provisioning should occur. For example, on AWS, this could be an Elastic Block Store (EBS) storage class.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs    # For AWS
parameters:
  type: gp2                            # Storage type (e.g., SSD, magnetic)
```

##### 2. **PersistentVolumeClaim (PVC)**
When you create a PVC, it references the `StorageClass`, which triggers dynamic provisioning.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-dynamic
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: standard            # This PVC will use the 'standard' StorageClass
```

##### 3. **Pod Using the PVC**
The pod uses the PVC as usual. The storage will be dynamically created when the pod requests it.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-using-dynamic-pvc
spec:
  containers:
    - name: busybox
      image: busybox
      command: ['sh', '-c', 'echo Hello Kubernetes! > /mnt/data/hello.txt; sleep 3600']
      volumeMounts:
        - name: pvc-storage
          mountPath: /mnt/data
  volumes:
    - name: pvc-storage
      persistentVolumeClaim:
        claimName: pvc-dynamic
```

#### Steps in Dynamic Provisioning:
1. **User creates a PVC**: Refers to a specific storage class.
2. **Kubernetes checks the StorageClass**: Based on the `storageClassName`, it looks for the correct provisioner.
3. **Provisioner creates a PV dynamically**: The provisioner (AWS EBS, GCP PD, etc.) creates a volume based on the `StorageClass` parameters (size, type, etc.).
4. **PVC is bound to the dynamically created PV**: Once the volume is created, the PVC gets bound to the newly created PV.
5. **Pod uses the PV**: The pod can now mount the dynamically created storage.

#### Benefits of Dynamic Provisioning:
- **Automation**: No need to manually create PVs before creating PVCs.
- **Scalability**: Especially in cloud environments, dynamic provisioning allows automatic scaling and management of storage without manual intervention.
- **Flexible**: Dynamic provisioning supports a variety of storage backends (cloud-based, networked storage like NFS, etc.).

#### Supported Provisioners:
Dynamic provisioning works with different storage backends, depending on the cloud provider or storage type:
- **AWS**: `kubernetes.io/aws-ebs` (EBS volumes)
- **GCP**: `kubernetes.io/gce-pd` (Google Persistent Disks)
- **Azure**: `kubernetes.io/azure-disk`
- **NFS**: `nfs-provisioner`
- And more.

Dynamic provisioning simplifies storage management in Kubernetes by automating the creation of volumes when they are needed, rather than requiring them to be manually set up ahead of time.