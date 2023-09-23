**Setting Up Dynamic NFS Storage Provisioning in OpenShift**

Tekton and Jenkins both require dynamic storage classes for persistent storage because they are used in OpenShift environments, and dynamic storage provisioning is essential for managing storage resources efficiently in such environments.

In openshift, storage classes are used to define the different types of storage that can be dynamically provisioned by the cluster. A storage class abstracts the underlying storage infrastructure, allowing administrators to specify storage requirements without being tied to a specific provider or implementation. When a storage class is defined, openshift can automatically provision storage volumes based on the defined class when requested by applications or workloads.

Tekton and Jenkins are both CI/CD tools that are widely used in openshift environments. These tools often require persistent storage to store build artifacts, logs, configurations, and other data that need to be retained across job executions and even across pod restarts.

Dynamic volume provisioning automates volume creation in Kubernetes, reducing manual efforts. It requires a storage class with a provisioner, enabling communication with the backend storage. Kubernetes has internal and external provisioners, such as NFS, which can be integrated for dynamic storage provisioning. This article demonstrates configuring the NFS external provisioner for dynamic storage in Kubernetes.


Presented below is a procedure detailing the configuration of the NFS external provisioner and the illustration of dynamic storage provisioning through the utilization of the NFS backend StorageClass.

Prerequisites:
OpenShift Cluster
NFS server to share the NFS shares


To create an NFS mount in RHEL (Red Hat Enterprise Linux) and mount it to the directory `/mnt/k8s_nfs_storage`, you'll need to follow these detailed steps:

1. **Install NFS Utilities (if not already installed):**
   Check if NFS utilities are installed on your system. If not, install them using the package manager (yum or dnf).
   ```bash
   sudo yum install nfs-utils -y
   ```

2. **Create the Mount Point Directory:**
   Ensure the directory `/mnt/k8s_nfs_storage` exists or create it if not.
   ```bash
   sudo mkdir -p /mnt/k8s_nfs_storage
   sudo chmod 777 /mnt/k8s_nfs_storage
   sudo chown -R nobody:nobody /mnt/k8s_nfs_storage
   sudo semanage fcontext -a -t public_content_rw_t "/mnt/k8s_nfs_storage(/.*)?"
   sudo restorecon -Rv /mnt/k8s_nfs_storage
   ```

3. **Configure the NFS Server:**
   On the NFS server (the machine with the shared directory), you need to configure the NFS export.
   Add a line in the `/etc/exports` file to specify the directory to be shared and the client(s) that are allowed to access it.
   Optional: *Replace `<client-ip>` with the IP address or subnet of the machine that will mount the NFS share.*
      ```
      echo "/mnt/k8s_nfs_storage *(rw,sync,no_subtree_check)" | sudo tee -a /etc/exports
      ```

      - `/NFS_EXPORT/to/shared/directory`: The directory you want to share via NFS.
      - `<client-ip>`: The IP address or subnet of the machine that will mount the NFS share.
      - `rw`: Read-Write access.
      - `sync`: Synchronous writes (safer but potentially slower).
      - `no_subtree_check`: Avoid subtree checking for better performance and flexibility.

   

5. **Export the NFS Share:**
   After making changes to the `/etc/exports` file, export the shared directory.
   ```
   sudo exportfs -rav
   ```

6. **Start NFS Services:**
   Make sure the NFS server services are running and enabled to start at boot.
   ```bash
   sudo systemctl enable nfs-server
   sudo systemctl start nfs-server
   ```

7. **Configure Firewall (if enabled):**
   If the NFS server has a firewall enabled, you need to allow NFS traffic.
   ```bash
   sudo firewall-cmd --add-service=nfs --add-service=rpc-bind --add-service=mountd --permanent
   sudo firewall-cmd --reload
   ```


8. **Test the NFS client – OCP Masters & worker:**
   Once you have the NFS server ready, you can configure the NFS client and test the filesystem access.
   ```
   [core@bastion]$ oc get nodes
   NAME       STATUS   ROLES                  AGE   VERSION
   master-0   Ready    control-plane,master   17d   v1.26.6+f245ced
   master-1   Ready    control-plane,master   17d   v1.26.6+f245ced
   master-2   Ready    control-plane,master   17d   v1.26.6+f245ced
   worker-0   Ready    worker                 17d   v1.26.6+f245ced
   worker-1   Ready    worker                 17d   v1.26.6+f245ced
   worker-2   Ready    worker                 17d   v1.26.6+f245ced
   ```
   ```
   [core@bastion]$ hostname -I
   10.0.90.24 2620:52:0:4ad0:f816:3eff:fe46:d297 
   ```
   ```
   [core@bastion]$ oc debug node/worker-0
   Temporary namespace openshift-debug-r8659 is created for debugging node...
   Starting pod/worker-0babbarlabpsipnq2redhatcom-debug ...
   To use host binaries, run `chroot /host`
   Pod IP: 10.74.210.229
   If you don't see a command prompt, try pressing enter.
   sh-4.4# chroot /host
   sh-5.1# showmount -e 10.0.90.24
   Export list for 10.0.90.24:
   /mnt/k8s_nfs_storage *
   sh-5.1# mkdir /mnt/test ; mount -t nfs 10.0.90.24:/mnt/k8s_nfs_storage /mnt/test
   sh-5.1# touch /mnt/test/testfile
   sh-5.1# umount /mnt/test
   sh-5.1# exit
   ```


**NFS Subdirectory External Provisioner in OpenShift:**
The NFS Subdirectory External Provisioner is a component that enables dynamic provisioning of PersistentVolumeClaims (PVCs) in OpenShift (and Kubernetes) using existing NFS shares as the underlying storage. It simplifies the process of setting up and managing NFS-based storage for your applications running in an OpenShift cluster.

1. Clone this repository:
   ```
   git clone https://github.com/BabbarPB08/NFS-OCP.git && cd NFS-OCP
   ```

3. Check the NFS export and set the variable for IP address and for NFS_EXPORT.
   ```
   IP=`hostname -I | awk '{ print $1 }'`
   NFS_EXPORT=`showmount -e $IP --no-headers | awk '{ print $1 }'`
   ```
   
4. Verify the values.
   ```
   echo $IP $NFS_EXPORT
   ```
   
6. Create a new namespace and install the NFS Subdirectory External Provisioner:
   ```
   oc new-project nfs-subdir-external-provisioner
   sed -i "s-<NFS_EXPORT>-$NFS_EXPORT-g" ./objects/deployment.yaml
   sed -i "s/<IP>/$IP/g" ./objects/deployment.yaml
   ```

7. Set the subject of the RBAC objects to the current namespace where the provisioner is being deployed
   ```
   NAMESPACE=`oc project -q`
   sed -i'' "s/namespace:.*/namespace: $NAMESPACE/g" ./objects/rbac.yaml ./objects/deployment.yaml
   oc create -f objects/rbac.yaml
   oc adm policy add-scc-to-user hostmount-anyuid system:serviceaccount:$NAMESPACE:nfs-client-provisioner   
   ```

8. Create the supporting objects and `storageClass`
   ```
   oc apply -f ./objects/deployment.yaml
   oc apply -f ./objects/class.yaml
   ```

7. Check the deployment status:
   ```
   oc get all -n nfs-subdir-external-provisioner
   ```

8. List the newly created StorageClass:
   ```
   oc get storageclass
   oc get storageclass nfs-client -o yaml
   ```

Test NFS Subdirectory External Provisioner:

1. Create a PVC and check its status:
   ```
   oc create -f ./objects/test-claim.yaml
   oc get pvc test-claim
   ```

2.  Create a PVC and check its status, PVC will be attached to it
   ```
   oc create -f ./objects/test-pod.yaml
   oc describe pod test-pod
   ```

You have now successfully deployed NFS Subdirectory External Provisioner in OpenShift to use NFS shares for persistent storage. The provisioner enables dynamic storage provisioning by creating PersistentVolumeClaims and binding them to existing NFS shares that are accessible to all OpenShift nodes.

Now that the NFS Subdirectory External Provisioner setup is complete, you can proceed with configuring the Tekton or Jenkins pipeline. This will allow you to integrate your applications with the provisioned NFS storage and automate your CI/CD workflows effectively.

To implement Tekton, you can follow the instructions provided in this [OpenShift repository](https://github.com/openshift/pipelines-tutorial)

Here is a demonstration of how to quickly deploy Jenkins using NFS storage:

1. Begin by creating a namespace for Jenkins.
```
oc new-project jenkins
```

2. Next, create a Persistent Volume Claim (PVC) to connect Jenkins pods with a specified storage class.
```
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    openshift.io/generated-by: OpenShiftNewApp
  finalizers:
  - kubernetes.io/pvc-protection
  labels:
    app: jenkins-persistent
    template: jenkins-persistent-template
  name: jenkins
  namespace: jenkins
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  volumeMode: Filesystem
  storageClassName: nfs-client # Specify the storage class name here
EOF
```

3. Now proceed to deploy the Jenkins application.
```
oc new-app jenkins-persistent
```

4. Once the application has been successfully deployed, you can access its URL.
```
oc get routes -o json -n jenkins | jq -r '.items[0].spec.host'
```
