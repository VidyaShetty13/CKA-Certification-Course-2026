# storage

## PV scenarios

<details>
  <summary>Create PV without specifying the hostPath.type </summary>

  ```yaml
  apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: web-assets-pv
  spec:
    capacity:
      storage: 256Mi
    volumeMode: Filesystem
    accessModes:
      - ReadWriteOnce
    persistentVolumeReclaimPolicy: Retain
    storageClassName: manual
    hostPath:
      path: /mnt/web-data
  ```
  - This will successfully create a PV, but there is no directory created in any node as no PVC is bound yet
    ```yaml
    Name:            web-assets-pv
    Labels:          <none>
    Annotations:     <none>
    Finalizers:      [kubernetes.io/pv-protection]
    StorageClass:    manual
    Status:          Available
    Claim:
    Reclaim Policy:  Retain
    Access Modes:    RWO
    VolumeMode:      Filesystem
    Capacity:        256Mi
    Node Affinity:   <none>
    Message:
    Source:
        Type:          HostPath (bare host directory volume)
        Path:          /mnt/web-data
        HostPathType:
    Events:            <none>
    
    ```
  - Next, lets create PVC
    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: web-assets-pvc
      namespace: default
    spec:
      accessModes:
        - ReadWriteOnce
      volumeMode: Filesystem
      resources:
        requests:
          storage: 200Mi
      storageClassName: manua 
    ```
    ```yaml
    Name:          web-assets-pvc
    Namespace:     default
    StorageClass:  manual
    Status:        Bound
    Volume:        web-assets-pv
    Labels:        <none>
    Annotations:   pv.kubernetes.io/bind-completed: yes
                   pv.kubernetes.io/bound-by-controller: yes
    Finalizers:    [kubernetes.io/pvc-protection]
    Capacity:      256Mi
    Access Modes:  RWO
    VolumeMode:    Filesystem
    Used By:       <none>
    Events:        <none>
    
    ```
  - Note: Even after pvc is bounded in the node there is no mountPoint yet created
    ```yaml
    $ docker exec -it b2ac95979ad6  bash
    root@kind-worker2:/# ls /mnt/
    root@kind-worker2:/# exit
    exit
    
    $ docker exec -it 0b162e3e3f31 bash
    root@kind-worker:/# ls /mnt/
    root@kind-worker:/# exit
    exit 
    ```
  - Next, create a Pod which uses the PVC
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      labels:
        run: web-server
      name: web-server
    spec:
      containers:
      - image: nginx
        name: web-server
        volumeMounts:
          - name: local-pvc
            mountPath: /usr/share/nginx/html
      volumes:
        - name: local-pvc
          persistentVolumeClaim:
            claimName: web-assets-pvc    
    ```
    ```yaml
    $ k get pods -owide
    NAME         READY   STATUS    RESTARTS   AGE   IP            NODE          NOMINATED NODE   READINESS GATES
    web-server   1/1     Running   0          21s   10.244.1.10   kind-worker   <none>           <none>
    ```
  - Verify if the hostPath is now created in the kind-worker node where the pod resides
    ```yaml
    $ docker ps
    CONTAINER ID   IMAGE                  COMMAND                  CREATED       STATUS          PORTS                       NAMES
    b2ac95979ad6   kindest/node:v1.34.3   "/usr/local/bin/entr…"   13 days ago   Up 47 minutes                               kind-worker2
    0b162e3e3f31   kindest/node:v1.34.3   "/usr/local/bin/entr…"   13 days ago   Up 47 minutes                               kind-worker
    c9699b4c2f10   kindest/node:v1.34.3   "/usr/local/bin/entr…"   13 days ago   Up 47 minutes   127.0.0.1:43897->6443/tcp   kind-control-plane

    # Exec into kind-worker node
    $ docker exec -it 0b162e3e3f31 bash
    root@kind-worker:/# ls /mnt/
    web-data
    root@kind-worker:/# ls /mnt/web-data/
    root@kind-worker:/# ls -ltr /mnt
    total 4
    drwxr-xr-x 2 root root 4096 Feb 10 09:41 web-data
    ```
  - Next, delete the pod, pvc, and PV
    Note: PV can be deleted only if PVC is deleted. PVC can be deleted only if pod is deleted
    ```yaml
    $ k delete -f pod.yaml -f pvc.yaml  -f pv.yaml
    pod "web-server" deleted from default namespace
    persistentvolumeclaim "web-assets-pvc" deleted from default namespace
    persistentvolume "web-assets-pv" deleted
    ```
  - Even after deleting the PV the hostPath will still have the /mnt/web-data. because the `persistentVolumeReclaimPolicy was set to Retain`
    ```yaml
    $ docker exec -it 0b162e3e3f31 bash
    root@kind-worker:/# ls -ltr /mnt
    total 4
    drwxr-xr-x 2 root root 4096 Feb 10 09:41 web-data 
    ```
  - You need to manually delete the directory from the node
    
</details>

<details>
  <summary>Why do we need to mention the **storageName** for the static PersistentVolume created manually</summary>

  - In Kubernetes, we use the storageClassName in a static PV for two primary reasons: Binding Control and Avoiding the Default Provisione
    - 1. Preventing "Default" Hijacking
      <br>"If a cluster has a Default StorageClass configured, any PVC that doesn't specify a class will automatically try to provision a new dynamic volume. By explicitly labeling a manual PV with a class name (like manual or static), and matching that in the PVC, we ensure the PVC binds to our specific pre-created volume instead of triggering the creation of an unnecessary cloud disk."
    - 2. Using it as a Matchmaker (The "Tagging" Logic)
      <br> "Even if a StorageClass object doesn't exist in the cluster, the storageClassName field acts as a filter. It creates a strong link between a PV and a PVC. The PersistentVolume controller will only bind a PVC to a PV if their class names match. It’s essentially a way to group static storage so that a 'Gold' PVC doesn't accidentally grab a 'Bronze' PV."
    - 3. Disabling Dynamic Provisioning
      <br> If we want to ensure a PVC only looks for static volumes and never uses a dynamic provisioner, we set the storageClassName to an empty string (""). This tells Kubernetes: 'Do not use any StorageClass; only look for PVs that also have an empty class name.

</details>


<details>
  <summary>how to disable dynamic provisioning </summary>

  - Disabling for a specific PVC (The "Static-Only" Mode)
    - If your cluster has a default StorageClass (like gp2 or standard) but you want to force a PVC to only bind to a manually created PV, you must use the Empty String Trick.
    - By setting storageClassName: "" (an empty string), you tell the PVC to ignore all dynamic provisioners and only look for PVs that also have an empty or "" storage class.
    - ```yaml
      apiVersion: v1
      kind: PersistentVolume
      metadata:
        name: web-assets-pv
      spec:
        capacity:
          storage: 256Mi
        volumeMode: Filesystem
        accessModes:
          - ReadWriteOnce
        persistentVolumeReclaimPolicy: Delete
        hostPath:
          path: /tmp/web-data
      ```
      ```yaml
      apiVersion: v1
      kind: PersistentVolumeClaim
      metadata:
        name: web-assets-pvc
        namespace: default
      spec:
        accessModes:
          - ReadWriteOnce
        volumeMode: Filesystem
        resources:
          requests:
            storage: 200Mi
        storageClassName: ""
      ```
      ```yaml
      apiVersion: v1
      kind: Pod
      metadata:
        labels:
          run: web-server
        name: web-server
      spec:
        containers:
        - image: nginx
          name: web-server
          volumeMounts:
            - name: local-pvc
              mountPath: /usr/share/nginx/html
        volumes:
          - name: local-pvc
            persistentVolumeClaim:
              claimName: web-assets-pvc
      ```
      ```yaml
      $ k describe pvc
      Name:          web-assets-pvc
      Namespace:     default
      StorageClass:
      Status:        Bound
      Volume:        web-assets-pv
      Labels:        <none>
      Annotations:   pv.kubernetes.io/bind-completed: yes
                     pv.kubernetes.io/bound-by-controller: yes
      Finalizers:    [kubernetes.io/pvc-protection]
      Capacity:      256Mi
      Access Modes:  RWO
      VolumeMode:    Filesystem
      Used By:       web-server
      Events:        <none>
      ```
      

  - Disabling the Cluster-Wide "Default"
    - If you want to stop the cluster from automatically using a specific StorageClass when a user forgets to define one, you must remove its "Default" status.
    - Identify the default: kubectl get sc (Look for the one marked (default)).
    - Remove the annotation: You don't delete the StorageClass; you just change its metadata so the "is-default" flag is false.<br>
      kubectl annotate storageclass <sc-name> storageclass.kubernetes.io/is-default-class-

  - The "Manual" Workaround
    - As we discussed earlier, you can also "disable" the logic by using a dummy name. If you create a PV and PVC with storageClassName: manual-only, and there is no actual StorageClass object named manual-only, the system has no provisioner to call. It effectively forces the cluster into "Static Mode."  
</details>

## PVC scenarios

## storage class scenarios

## Errors

<details>
  <summary>Message:         host_path deleter only supports /tmp/.+ but received provided /mnt/web-data</summary>

  ```yaml
  apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: web-assets-pv
  spec:
    capacity:
      storage: 256Mi
    volumeMode: Filesystem
    accessModes:
      - ReadWriteOnce
    persistentVolumeReclaimPolicy: Delete  ---> this is the cause for the violation. if it was Retain it would have helped
    hostPath:
      path: /mnt/web-data
  ```
  - It is telling you that the Provisioner (the software responsible for managing the volumes) has a strict whitelist of where it is allowed to delete files.
  - The Rule: The provisioner is configured to only manage and delete files inside the /tmp/ directory (specifically anything matching the regex /tmp/.+).
  - The Violation: You provided /mnt/web-data
  - The Consequence: Because the provisioner doesn't "trust" that path, it refuses to handle the lifecycle (specifically the deletion/cleanup) of that volume.
</details>
