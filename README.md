# How to Setup Dynamic NFS Provisioning in a Kubernetes Cluster

Dynamic NFS storage provisioning in Kubernetes streamlines the creation and management of NFS volumes for your Kubernetes applications. It eliminates the need for manual intervention or pre-provisioned storage. The NFS provisioner dynamically creates persistent volumes (PVs) and associates them with persistent volume claims (PVCs), making the process more efficient. If you have an external NFS share and want to use it in a pod or deployment, the nfs-subdir-external-provisioner provides a solution for effortlessly setting up a storage class to automate the management of your persistent volumes.

## Prerequisites
 - Pre-installed Kubernetes Cluster - Minikube
 - A Regular user which has admin rights on the Kubernetes cluster
 - Internet Connectivity

### Step 1: Installing the NFS Server
```
sudo apt-get update
sudo apt-get install nfs-common nfs-kernel-server -y
```

Create a directory to export:

```
sudo mkdir -p /mnt/nfs_share
sudo chown nobody:nogroup /mnt/nfs_share
sudo chmod 2770 /mnt/nfs_share
```

Export directory

```
sudo vi /etc/exports
add below line to this file
/mnt/nfs_share *(rw,sync,no_subtree_check,no_root_squash,insecure)
execute below
sudo exportfs -av
verify response as below
exporting *:/mnt/nfs_share
Restart nfs server
sudo systemctl restart nfs-kernel-server
sudo systemctl status nfs-kernel-server
Check the mount
/sbin/showmount -e
It should show something
Export list for skjpune:
/mnt/nfs_share *
```

### Step 2: Install Helm
Follow this link 
https://helm.sh/docs/intro/install/

Verify Helm Installation
```
helm version
Verify Response
version.BuildInfo{Version:"v3.15.4", GitCommit:"fa9efb07d9d8debbb4306d72af76a383895aa8c4", GitTreeState:"clean", GoVersion:"go1.22.6"}
```

### Step 3: Add Helm Repo
```
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner
```

### Step 4: Get NFS Server IP
 - On minukube server 
   ```
   login to minikube node 
   minikube ssh
   execute command
   ip r | grep default
   response would be
   default via 192.168.49.1 dev eth0
   here 192.168.49.1 is the minikube gateway ip which we would use as nfs server ip
   ```

### Step 5: Create Kubernetes Namespace (Optional)
```
kubectl create namespace nfs-provisioner
```

### Step 6: Install Helm Chart for NFS
```
helm install nfs-subdir-external-provisioner \
nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
--namespace nfs-provisioner \
--set nfs.server=192.168.49.1 \
--set nfs.path=/data/nfs \
--set storageClass.onDelete=true
```

### Step 7: Check pods and storage classes
```
kubectl get pod
NAME                                               READY   STATUS    RESTARTS       AGE
nfs-subdir-external-provisioner-766d8bb659-895vf   1/1     Running   6 (148m ago)   23h

kubectl get sc
NAME                 PROVISIONER                                     RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-client           cluster.local/nfs-subdir-external-provisioner   Delete          Immediate           true                   23h
standard (default)   k8s.io/minikube-hostpath                        Delete          Immediate           false                  23h

```

### Step 7: Dynamic PVC Volume Create Testing
Now we can test creating dynamic PVC volume.

nfs-pvc.

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-client
  resources:
    requests:
      storage: 1Gi
```

Create an NGINX pod that mounts the NFS export in its web directory:

deployment.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nfs-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      volumes:
        - name: nfs-nginx
          persistentVolumeClaim:
            claimName: nfs-pvc
      containers:
        - image: nginx
          name: nginx
          volumeMounts:
            - name: nfs-nginx
              mountPath: /usr/share/nginx/html
```

```
kubectl apply -f nfs-pvc.yaml
kubectl apply -f deployment.yaml
```

Now, let’s enter the pod and create an ‘index.html’ file under ‘/usr/share/nginx/html.’

```
kubectl get po
NAME                                               READY   STATUS    RESTARTS       AGE
nfs-subdir-external-provisioner-766d8bb659-895vf   1/1     Running   6 (151m ago)   23h

kubectl exec -it nfs-subdir-external-provisioner-766d8bb659-895vf -- bash

#now we are into pod
# cd /usr/share/nginx/html
# ls -l
# echo "hello world" >index.html
# ls -l
# exit
```

```
cd /mnt
cd nfs_share
ls -l
total 20
drwxrwsrwx 5 nobody nogroup 4096 Aug 24 17:28 ./
drwxr-xr-x 6 root   root    4096 Aug 23 18:03 ../
drwxrwxrwx 2 root   nogroup 4096 Aug 24 17:28 nfs-provisioner-nfs-pvc-pvc-24ef7834-f985-442a-a6f5-0835ee29d3b5/

cd nfs-provisioner-nfs-pvc-pvc-24ef7834-f985-442a-a6f5-0835ee29d3b5
ls -l
you should see index.html there
```
