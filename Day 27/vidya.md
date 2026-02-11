# storage

## Extra questions

<details>
  <summary>For a hostPath volume, if the node itself goes down, what happens to the data?</summary>

- The data remains on the physical disk of that specific machine. However, because hostPath is not network storage, Kubernetes cannot "move" that data to a different node.
- If the Pod is rescheduled to a new node, it will start with an empty directory (or whatever is at that path on the new node). The original data is "trapped" on the offline node.
  
</details>

<details>
  <summary>What happens if two containers in the same pod write to the same PVC?</summary>

  - Both containers can write simultaneously. Kubernetes does not manage "file locking."
  - f they write to the same file, they will likely corrupt it or overwrite each other’s data. If they write to different files in the same volume, it works perfectly (this is a common way to use "Sidecar" containers).
    
</details>

<details>
  <summary>If a PVC requests 5GB and is bound to a 7GB PV, can another PVC use the remaining 2GB?</summary>

  - No, In Kubernetes, the binding between a PVC and a PV is 1-to-1. Once a PV is "Bound," it is fully consumed by that specific PVC, even if the PVC only uses a fraction of the available space. The remaining 2GB is "lost"/wasted.
    
</details>

<details>
  <summary>What happens if I try to delete a PV before the PVC or Pod?</summary>

  - The PV will enter the Terminating state but will not disappear.
  - This is due to Storage Object Protection. The kubernetes.io/pv-protection finalizer ensures a PV is not deleted while it is still bound to a PVC. Once you delete the PVC, the PV will finally vanish.
    
</details>

<details>
  <summary>When a PV is created, is it available on all nodes?</summary>

  - The PV object is cluster-scoped (visible to all), but the actual storage may not be.
  - Cloud/Network PVs (EBS, EFS, NFS): Effectively available to all nodes
  - Local/hostPath PVs: Physically exist on one node only. Even though you can see the "object" from any node, only the specific node with the disk can actually mount it.
</details>

<details>
  <summary>What happens if a Pod is created on Node A, but the PV (Immediate binding) is on Node B?</summary>
  - The Pod will stay in ContainerCreating (not Pending) and eventually fail with a FailedMount error.
  - With VolumeBindingMode: Immediate, the PVC binds to the PV before the Pod is even scheduled. If the Scheduler then accidentally places the Pod on a node that can't reach that specific PV (like a hostPath on a different node), the Pod is stuck.
  - This is why we use WaitForFirstConsumer—it tells K8s: "Don't bind the storage until you know exactly where the Pod is going to land."
</details>


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
      storageClassName: manual
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
  <summary>Why do we need to mention the **storageClassName** for the static PersistentVolume created manually</summary>

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

<details>
  <summary>We are deploying a application in a multi-node cloud environment. Why would we choose WaitForFirstConsumer over Immediate as the volume binding mode in our StorageClass? - HARD rule</summary>

  1. Create a PV tied to a specific Node
     - We will use nodeAffinity on the PV. This tells K8s: "This volume only exists on kind-control-plane."
       ```yaml
        apiVersion: v1
        kind: PersistentVolume
        metadata:
          name: node-locked-pv
        spec:
          capacity:
            storage: 1Gi
          accessModes:
            - ReadWriteOnce
          storageClassName: manual
          hostPath:
            path: /tmp/data
          nodeAffinity:
            required:
              nodeSelectorTerms:
              - matchExpressions:
                - key: kubernetes.io/hostname
                  operator: In
                  values:
                  - kind-control-plane  # This locks the PV to the control-plane node       
       ```
    
       
  2. Create the PVC
     - We use Immediate binding (implicit here since we aren't using a StorageClass object with a policy, but we'll bind it manually).
       ```yaml
        apiVersion: v1
        kind: PersistentVolumeClaim
        metadata:
          name: locked-pvc
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 100Mi
          storageClassName: manual       
       ```
      
  3.  Create the Pod on a DIFFERENT Node
      - Now, we force the Pod to run on a worker node (e.g., kind-worker) while using the PVC that is locked to the control-plane.
        ```yaml
          apiVersion: v1
          kind: Pod
          metadata:
            name: conflicted-pod
          spec:
            nodeName: kind-worker # This forces the pod to the worker node
            containers:
            - name: nginx
              image: nginx
              volumeMounts:
              - name: data
                mountPath: /data
            volumes:
            - name: data
              persistentVolumeClaim:
                claimName: locked-pvc        
        ```
      
     
  4. What will happen?
     - The PVC will bind to the PV because the storageClassName and size match.
     - The Pod will be forced onto kind-worker
     - The Error: The Pod will stay in ContainerCreating.
       ```yaml
        $ k get pods -owide
        NAME             READY   STATUS              RESTARTS   AGE     IP       NODE          NOMINATED NODE   READINESS GATES
        conflicted-pod   0/1     ContainerCreating   0          5m10s   <none>   kind-worker   <none>           <none>
        
        $ k describe pod conflicted-pod
        Name:             conflicted-pod
        Namespace:        default
        Priority:         0
        Service Account:  default
        Node:             kind-worker/172.19.0.2
        Start Time:       Tue, 10 Feb 2026 14:06:43 +0300
        Labels:           <none>
        Annotations:      <none>
        Status:           Pending
        IP:
        IPs:              <none>
        Containers:
          nginx:
            Container ID:
            Image:          nginx
            Image ID:
            Port:           <none>
            Host Port:      <none>
            State:          Waiting
              Reason:       ContainerCreating
            Ready:          False
            Restart Count:  0
            Environment:    <none>
            Mounts:
              /data from data (rw)
              /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-7hvzm (ro)
        Conditions:
          Type                        Status
          PodReadyToStartContainers   False
          Initialized                 True
          Ready                       False
          ContainersReady             False
          PodScheduled                True
        Volumes:
          data:
            Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
            ClaimName:  locked-pvc
            ReadOnly:   false
          kube-api-access-7hvzm:
            Type:                    Projected (a volume that contains injected data from multiple sources)
            TokenExpirationSeconds:  3607
            ConfigMapName:           kube-root-ca.crt
            Optional:                false
            DownwardAPI:             true
        QoS Class:                   BestEffort
        Node-Selectors:              <none>
        Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                                     node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
        Events:
          Type     Reason       Age                   From     Message
          ----     ------       ----                  ----     -------
          Warning  FailedMount  74s (x10 over 5m15s)  kubelet  MountVolume.NodeAffinity check failed for volume "node-locked-pv" : no matching NodeSelectorTerms
               
       ```
</details>

<details>
  <summary>SOFT rule: Create a PV with no nodeAffinity rules set to restrict to particular node. Then create 2 pods on different nodes and see what is the status of those 2 pods</summary>

  - Create a PV and PVC
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
        path: /tmp/web-data
    ---
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
      storageClassName: "manual"
    
    ```
    
  - Create 2 pods on 2 different nodes
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      labels:
        run: try
      name: try
    spec:
      containers:
      - image: nginx
        name: try
        volumeMounts:
          - name: pvc-volume
            mountPath: /usr/share/nginx/html
      volumes:
        - name: pvc-volume
          persistentVolumeClaim:
            claimName: web-assets-pvc
    
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      labels:
        run: try-2
      name: try-2
    spec:
      nodeName: kind-worker2
      containers:
      - image: nginx
        name: try-2
        volumeMounts:
          - name: pvc-volume
            mountPath: /usr/share/nginx/html
      volumes:
        - name: pvc-volume
          persistentVolumeClaim:
            claimName: web-assets-pvc
    
    ```
    
  - Check 2 pods are running
    ```yaml
    $ k get pods -owide
    NAME    READY   STATUS    RESTARTS   AGE   IP            NODE           NOMINATED NODE   READINESS GATES
    try     1/1     Running   0          21s   10.244.1.32   kind-worker    <none>           <none>
    try-2   1/1     Running   0          21s   10.244.2.11   kind-worker2   <none>           <none>
    ```
    
  - Create a file on each container , and verify if its present on both the nodes
    ```yaml
    $ k exec -it try -- bash
    root@try:/# touch /usr/share/nginx/html/pod1
    root@try:/# exit
    
    exit$ k exec -it try-2 -- bash
    root@try-2:/# touch /usr/share/nginx/html/pod2
    root@try-2:/# exit
    exit
    
    $ docker ps
    CONTAINER ID   IMAGE                  COMMAND                  CREATED       STATUS       PORTS                       NAMES
    b2ac95979ad6   kindest/node:v1.34.3   "/usr/local/bin/entr…"   2 weeks ago   Up 3 hours                               kind-worker2
    0b162e3e3f31   kindest/node:v1.34.3   "/usr/local/bin/entr…"   2 weeks ago   Up 3 hours                               kind-worker
    
    $ docker exec -it b2ac95979ad6 bash
    root@kind-worker2:/# ls -ltr /tmp/web-data/
    total 0
    -rw-r--r-- 1 root root 0 Feb 10 12:26 pod2
    root@kind-worker2:/# exit
    exit
    
    $ docker exec -it 0b162e3e3f31 bash
    root@kind-worker:/# ls -ltr /tmp/web-data/
    total 0
    -rw-r--r-- 1 root root 0 Feb 10 12:26 pod1
    root@kind-worker:/# exit
    exit

    ```
  - Results:
    - How K8s sees it: "Here is a volume that exists at /tmp/web-data. I have no rules telling me it only exists on one node."
    - Why it didn't block it: The standard hostPath plugin is "dumb" because it doesn't verify if that path is the same physical drive across nodes. It simply tells the Kubelet on whatever node the Pod lands on: "Hey, mount /tmp/web-data from your local disk."
    - The "Ghost" Result: K8s allowed it, but as you saw, Node A had its own /tmp/web-data and Node B had its own. They were not sharing data. They were just two pods happening to use the same name for different local folders.
    - 
</details>

<details>
  <summary> When would RWX + nodeAffinity actually be useful?</summary>
  - You would use this if you have multiple Pods on the same specific node that all need to write to the same folder.
  - Example: You have a massive 128-core "Super Node" with a specialized NVMe drive. You want 10 replicas of your app to all run on that node and all write to that drive.
  - The nodeAffinity ensures they all land on the Super Node.
  - The RWX ensures they can all open the same directory for writing.
</details>

<details>
  <summary> Bind a new PVC onto to the Released PV and see if it works or not</summary>

  - Create a PV, PVC and pod
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
        path: /tmp/web-data
    ---
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
      storageClassName: "manual"
    ---
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
    
  - status pod pvc,pv,pod
    ```yaml
    NAME         READY   STATUS    RESTARTS   AGE   IP            NODE          NOMINATED NODE   READINESS GATES
    web-server   1/1     Running   0          4s    10.244.1.21   kind-worker   <none>           <none>

    Name:            web-assets-pv
    Labels:          <none>
    Annotations:     pv.kubernetes.io/bound-by-controller: yes
    Finalizers:      [kubernetes.io/pv-protection]
    StorageClass:    manual
    Status:          Bound
    Claim:           default/web-assets-pvc
    Reclaim Policy:  Retain
    Access Modes:    RWO
    VolumeMode:      Filesystem
    Capacity:        256Mi
    Node Affinity:   <none>
    Message:
    Source:
        Type:          HostPath (bare host directory volume)
        Path:          /tmp/web-data
        HostPathType:
    Events:            <none>

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
    Used By:       web-server
    Events:        <none>
    
    ```
    
  - Now create a new file inside the pod mount directory
    ```yaml
    $ k exec -it web-server -- bash
    root@web-server:/# echo "newfile">> /usr/share/nginx/html/nginx.txt
    root@web-server:/# cat /usr/share/nginx/html/nginx.txt
    newfile 
    ```
    
  - verify the file exists in the hostnode
    ```yaml
    $ docker exec -it 0b162e3e3f31 bash
    root@kind-worker:/# cd /tmp/
    root@kind-worker:/tmp# ls
    web-data
    root@kind-worker:/tmp# cd web-data/
    root@kind-worker:/tmp/web-data# ls
    nginx.txt  
    ```
    
  - Delete the pod, and pvc
    ```yaml
    pod "web-server" deleted from default namespace
    persistentvolumeclaim "web-assets-pvc" deleted from default namespace
    ```
    
  - check the PV status
    ```yaml
    Name:            web-assets-pv
    Labels:          <none>
    Annotations:     pv.kubernetes.io/bound-by-controller: yes
    Finalizers:      [kubernetes.io/pv-protection]
    StorageClass:    manual
    Status:          Released
    Claim:           default/web-assets-pvc
    Reclaim Policy:  Retain
    Access Modes:    RWO
    VolumeMode:      Filesystem
    Capacity:        256Mi
    Node Affinity:   <none>
    Message:
    Source:
        Type:          HostPath (bare host directory volume)
        Path:          /tmp/web-data
        HostPathType:
    Events:            <none>

    $ k get pv
    NAME            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                    STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
    web-assets-pv   256Mi      RWO            Retain           Released   default/web-assets-pvc   manual         <unset>                          3m58s

    ```
    
  - Attach a new pvc to the same PV
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
      storageClassName: "manual"    
    ```
    
  - check the pvc status
    ```yaml
    $ k get pvc
    NAME             STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
    web-assets-pvc   Pending                                      manual         <unset>                 2s
    
        $ k describe pvc
    Name:          web-assets-pvc
    Namespace:     default
    StorageClass:  manual
    Status:        Pending
    Volume:
    Labels:        <none>
    Annotations:   <none>
    Finalizers:    [kubernetes.io/pvc-protection]
    Capacity:
    Access Modes:
    VolumeMode:    Filesystem
    Used By:       <none>
    Events:
      Type     Reason              Age   From                         Message
      ----     ------              ----  ----                         -------
      Warning  ProvisioningFailed  5s    persistentvolume-controller  storageclass.storage.k8s.io "manual" not found
    ```

  - Data still exists in the hostnode
    ```yaml
    $ docker exec -it 0b162e3e3f31 bash
    root@kind-worker:/# cd  /tmp/web-data/
    root@kind-worker:/tmp/web-data# lks
    bash: lks: command not found
    root@kind-worker:/tmp/web-data# ls
     f1.txt  'manual file'   nginx.txt
    root@kind-worker:/tmp/web-data# ls
     f1.txt  'manual file'   nginx.txt
    root@kind-worker:/tmp/web-data# ls -ltr
    total 4
    -rw-r--r-- 1 root root 8 Feb 10 11:21  nginx.txt
    -rw-r--r-- 1 root root 0 Feb 10 11:23  f1.txt
    -rw-r--r-- 1 root root 0 Feb 10 11:23 'manual file'
    root@kind-worker:/tmp/web-data#
    
    ```

  - Final thoughts
    
    1. Why the "StorageClass not found" error?
       Even though you are doing static provisioning, the PVC sees storageClassName: manual. It looks for a StorageClass object named manual to see if it should try to create a volume for you. Since that object doesn't exist, it throws a warning.
       
       However, that's not what's stopping the binding. The real reason is the PV Status.
       
    2. The "Released" Block
       It will show Status: Released.

        Here is why your new PVC is Pending:
        
        When a PV has a Retain policy and its associated PVC is deleted, the PV moves to Released.
        
        A Released PV is NOT Available.
        
        It still contains a "memory" of the old PVC in its configuration (the claimRef).
        
        Kubernetes refuses to bind a new PVC to a Released PV because it wants to protect the data. It assumes the Admin needs to check the data before making the volume "Available" for someone else.

    3. Solution
       - Edit the PV: kubectl edit pv <pv-name>
         - Find the claimRef block: It will look something like this:
         - Delete the entire claimRef section
         - Save and exit.
       - What happens next: As soon as you remove the claimRef, the PV status will flip from Released to Available. Since your new PVC is already sitting there in Pending, the controller will immediately pair them up. If you have not deleted the old content from the hostPath node then the new pods also will be able to see the older contents
         ```yaml
          $ k edit pv web-assets-pv
          persistentvolume/web-assets-pv edited
          
          $ k get pv
          NAME            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
          web-assets-pv   256Mi      RWO            Retain           Available           manual         <unset>                          17m
          
          $ k get pvc
          NAME             STATUS   VOLUME          CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
          web-assets-pvc   Bound    web-assets-pv   256Mi      RWO            manual         <unset>                 13m
          
          $ k get pv
          NAME            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
          web-assets-pv   256Mi      RWO            Retain           Bound    default/web-assets-pvc   manual         <unset>                          18m
          
          $ k create -f pod.yaml
          pod/web-server created
          
          $ k exec -it web-server -- bash
          root@web-server:/# cd /usr/share/nginx/html/
          root@web-server:/usr/share/nginx/html# ls
           f1.txt  'manual file'   nginx.txt        
         ```
          
</details>

<details>
  <summary> Try to bind 2 pvc onto same PV</summary>

  - create a pv
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
        path: /tmp/web-data
    
    ```
  - create 2 pvc on same yaml
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
      storageClassName: "manual"
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: web-assets-pvc-2
      namespace: default
    spec:
      accessModes:
        - ReadWriteOnce
      volumeMode: Filesystem
      resources:
        requests:
          storage: 200Mi
      storageClassName: "manual"
    
    ```
  - verify the status
    ```yaml
    kName:            web-assets-pv
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
        Path:          /tmp/web-data
        HostPathType:
    Events:            <none>
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
    
    
    Name:          web-assets-pvc-2
    Namespace:     default
    StorageClass:  manual
    Status:        Pending
    Volume:
    Labels:        <none>
    Annotations:   <none>
    Finalizers:    [kubernetes.io/pvc-protection]
    Capacity:
    Access Modes:
    VolumeMode:    Filesystem
    Used By:       <none>
    Events:
      Type     Reason              Age              From                         Message
      ----     ------              ----             ----                         -------
      Warning  ProvisioningFailed  0s (x2 over 4s)  persistentvolume-controller  storageclass.storage.k8s.io "manual" not found
        
    ```

    - One pvc will be successfully bounded to the PV
    - the another PVC will fail
</details>

<details>
  <summary>How do you change the Reclaim Policy of a PV that is already live/active?</summary>

  - By performing the patch on the Persistentvolume object
    ```yaml
    $ kubectl patch pv <name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}')
    ```
</details>

<details>
  <summary>How do I make an AWS EBS volume ReadWriteMany?</summary>
  - You can't." You would have to switch to a service like AWS EFS (NFS) to get RWX capabilities

</details>

<details>
  <summary>You have a PV that is marked as ReadWriteOnce. You have two different Pods (Pod A and Pod B). Both Pods use the same PVC.</summary>

  - Create a PV and PVC
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
        path: /tmp/web-data
    ---    
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
      storageClassName: "manual"
    
    ```
  - Create 2 pods
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      labels:
        run: try
      name: try
    spec:
      nodeName: kind-worker
      containers:
      - image: nginx
        name: try
        volumeMounts:
          - name: pvc-volume
            mountPath: /usr/share/nginx/html
      volumes:
        - name: pvc-volume
          persistentVolumeClaim:
            claimName: web-assets-pvc
    
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      labels:
        run: try-2
      name: try-2
    spec:
      nodeName: kind-worker
      containers:
      - image: nginx
        name: try-2
        volumeMounts:
          - name: pvc-volume
            mountPath: /usr/share/nginx/html
      volumes:
        - name: pvc-volume
          persistentVolumeClaim:
            claimName: web-assets-pvc
    
    ```
    
  - Check pod status
    ```yaml
    $ k get pods -owide
    NAME    READY   STATUS    RESTARTS   AGE     IP            NODE          NOMINATED NODE   READINESS GATES
    try     1/1     Running   0          3m37s   10.244.1.26   kind-worker   <none>           <none>
    try-2   1/1     Running   0          3m37s   10.244.1.27   kind-worker   <none>           <none>
        
    ```

  - Results
    - Once means "One Node," not "One Pod."
    - Case 1: Same Node (SUCCESS)
      Since the hardware (the volume) is plugged into the "motherboard" of that specific node, any number of Pods living on that node can share the mount point. The Linux kernel handles the file locking between the processes.
    - Case 2: Different Nodes (FAILURE)
      If the scheduler tries to put Pod B on Node 2, Pod B will get stuck in ContainerCreating. If you run kubectl describe pod, you will see a Multi-Attach error.
       
      Error: Multi-Attach error for volume "pv-name" Volume is already used by pod "pod-a" on node "node-1"
</details>

<details>
  <summary>what is RWOP.</summary>

  - Before ReadWriteOncePod existed, ReadWriteOnce (RWO) had a loophole: it allowed multiple Pods to write to the same disk as long as they were on the same node
  - If you use ReadWriteOncePod:
    - Only one Pod in the entire cluster can use that volume.
    - If a second Pod tries to start (even on the same node), Kubernetes will block it.
    - This is used for critical databases where even two local processes writing to the same data might cause corruption.
      
  - Create a PV and PVC with Accessmode ReadWriteOncePod
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
        - ReadWriteOncePod
      persistentVolumeReclaimPolicy: Retain
      storageClassName: manual
      hostPath:
        path: /tmp/once-pod
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: web-assets-pvc
      namespace: default
    spec:
      accessModes:
        - ReadWriteOncePod
      volumeMode: Filesystem
      resources:
        requests:
          storage: 200Mi
      storageClassName: "manual"
        
    ```
  - create a pod-1
    ```yaml
    $ cat pod-2.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: pod-1
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: web-assets-pvc
    
    ```

    ```yaml
    $ k get pods -owide
    NAME    READY   STATUS    RESTARTS   AGE     IP            NODE          NOMINATED NODE   READINESS GATES
    pod-1   1/1     Running   0          2m28s   10.244.1.33   kind-worker   <none>           <none>
    ```
    
  - create a conflicting pod-2
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: pod-2
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: web-assets-pvc
    
    ```
  - pod-2 will be in pending state with below warning
    ```yaml
    $ k get pods -owide
    NAME    READY   STATUS    RESTARTS   AGE     IP            NODE          NOMINATED NODE   READINESS GATES
    pod-1   1/1     Running   0          2m28s   10.244.1.33   kind-worker   <none>           <none>
    pod-2   0/1     Pending   0          115s    <none>        <none>        <none>           <none>
        
    ```
    ```yaml
    $ k describe pod pod-2
    Name:             pod-2
    Namespace:        default
    Priority:         0
    Service Account:  default
    Node:             <none>
    Labels:           <none>
    Annotations:      <none>
    Status:           Pending
    IP:
    IPs:              <none>
    Containers:
      nginx:
        Image:        nginx
        Port:         <none>
        Host Port:    <none>
        Environment:  <none>
        Mounts:
          /data from data (rw)
          /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-77bd8 (ro)
    Conditions:
      Type           Status
      PodScheduled   False
    Volumes:
      data:
        Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
        ClaimName:  web-assets-pvc
        ReadOnly:   false
      kube-api-access-77bd8:
        Type:                    Projected (a volume that contains injected data from multiple sources)
        TokenExpirationSeconds:  3607
        ConfigMapName:           kube-root-ca.crt
        Optional:                false
        DownwardAPI:             true
    QoS Class:                   BestEffort
    Node-Selectors:              <none>
    Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
    Events:
      Type     Reason            Age   From               Message
      ----     ------            ----  ----               -------
      Warning  FailedScheduling  9s    default-scheduler  0/3 nodes are available: 1 node(s) had untolerated taint(s), 2 node(s) unavailable due to PersistentVolumeClaim with ReadWriteOncePod access mode already in-use by another pod. no new claims to deallocate, preemption: 0/3 nodes are available: 1 Preemption is not helpful for scheduling, 2 No preemption victims found for incoming pod.
        
    ```
</details>

<details>
  <summary>How do you prevent a data corruption issue where a secondary Pod starts up before the first one has finished shutting down (like during a rolling update)?</summary>

  - ReadWriteOncePod. It ensures that at no point in time can two Pods ever touch that disk simultaneously
    
</details>

<details>
  <summary>We switched our Database Deployment to use ReadWriteOncePod for safety. Now, whenever we update the image, the new Pod stays 'Pending' forever and the old Pod never dies. What’s happening?</summary>

  - This is a Deployment Deadlock. By default, a Deployment starts a new Pod (maxSurge) before killing the old one. Because the PVC is RWOP, the scheduler sees the old Pod is still using the volume and refuses to start the new one. Since the new one can't start and pass health checks, the old one is never killed. To fix this, we must set the Deployment strategy to type: Recreate so the old Pod is killed first, releasing the volume for the new one.
    
</details>

<details>
  <summary>If I have a StatefulSet with 3 replicas and a volumeClaimTemplate using RWOP, will the pods start?</summary>

  - yes, they will. In a StatefulSet, each replica gets its own unique PVC. Since each PVC is only being used by one specific Pod (e.g., web-0 uses data-web-0), RWOP won't block them. RWOP only blocks multiple pods trying to use the same PVC
    
</details>
  
<details>
  <summary> can PVC ask for multiple accessmodes?</summary>

  - You can list multiple access modes in your PersistentVolumeClaim (PVC) definition, but the actual binding and usage depend heavily on what the storage provider supports and how the volume is ultimately mounted.
  - create pv and pvc
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
        - ReadWriteMany
      persistentVolumeReclaimPolicy: Retain
      storageClassName: manual
      hostPath:
        path: /tmp/once-pod
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: web-assets-pvc
      namespace: default
    spec:
      accessModes:
        - ReadWriteOnce
        - ReadWriteMany
      volumeMode: Filesystem
      resources:
        requests:
          storage: 200Mi
      storageClassName: "manual"
    ```
  - check the status of pv and pvc
    ```yaml
    $ k get pv,pvc
    NAME                             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
    persistentvolume/web-assets-pv   256Mi      RWO,RWX        Retain           Bound    default/web-assets-pvc   manual         <unset>                          21s
    
    NAME                                   STATUS   VOLUME          CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
    persistentvolumeclaim/web-assets-pvc   Bound    web-assets-pv   256Mi      RWO,RWX        manual         <unset>                 5s
    ---
    $ k get pvc -oyaml
    apiVersion: v1
    items:
    - apiVersion: v1
      kind: PersistentVolumeClaim
      metadata:
        annotations:
          pv.kubernetes.io/bind-completed: "yes"
          pv.kubernetes.io/bound-by-controller: "yes"
        creationTimestamp: "2026-02-10T12:50:34Z"
        finalizers:
        - kubernetes.io/pvc-protection
        name: web-assets-pvc
        namespace: default
        resourceVersion: "303799"
        uid: 7b238272-cfa9-495c-bdc3-f652904aebfd
      spec:
        accessModes:
        - ReadWriteOnce
        - ReadWriteMany
        resources:
          requests:
            storage: 200Mi
        storageClassName: manual
        volumeMode: Filesystem
        volumeName: web-assets-pv
      status:
        accessModes:
        - ReadWriteOnce
        - ReadWriteMany
        capacity:
          storage: 256Mi
        phase: Bound
    kind: List
    metadata:
      resourceVersion: ""

    ```
    
  - 2. How Binding Works
    Kubernetes treats the accessModes list as a set of requirements.
    
    The Match: The PVC will only bind to a PersistentVolume (PV) that has at least all the modes you listed.
    
    The Limitation: Even if a PV supports both, most storage backends (like AWS EBS or Azure Disk) only allow the volume to be mounted in one mode at a time for a specific Pod.

</details>

## storage class scenarios

<details>
  <summary>Can you expand a volume that is currently in use by a Pod?</summary>

  1. storgae class must have allowVolumeExpansion set to true
  2. Create a PVC with request capacity of 100Mi
  3. Create a pod. Pod will be in running state
  4. Now, change the PVC storage request from 100Mi to 200Mi and wait for the bvolume expanison to completed
  5. Once the volume is expanded you can verify if the pod data storage has increased by runnign df -h command

</details>


<details>
  <summary>What happens if you deploy a StorageClass with no-provisioner</summary>

  - If you deploy a StorageClass with provisioner: kubernetes.io/no-provisioner, you are essentially creating a StorageClass that cannot talk to any hardware.
  
  1. It disables "Dynamic" Provisioning
    - A StorageClass is usually a set of instructions for a plugin to go out and create a disk (like an AWS EBS volume). By using no-provisioner, you are telling Kubernetes: "I want the features of a StorageClass (like binding modes and reclaim policies), but I don't want you to automatically create any disks.

  2. The "Matching" Requirement
    - Since no disk will be created automatically, any PersistentVolumeClaim (PVC) that requests this StorageClass will stay in a Pending state forever UNLESS you (the Administrator) manually create a PersistentVolume (PV) that:
       - Lists that same storageClassName
       - Matches the PVC’s requirements (size, access modes).

  3. You use no-provisioner primarily for Local Persistent Volumes.
    
  
</details>

<details>
  <summary>Storage class with no Provisioner and the volumeBindingMode is set to Immediate</summary>

  - Create a storage class
    ```yaml
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: local-link
    provisioner: kubernetes.io/no-provisioner
    volumeBindingMode: Immediate
    ```
    ```yaml
    $ k get sc
    NAME                 PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
    local-link           kubernetes.io/no-provisioner   Delete          Immediate              false                  25s
    ```
  - Create a PVC
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
      storageClassName: "local-link"
    ```
    ```yaml
    Name:          web-assets-pvc
    Namespace:     default
    StorageClass:  local-link
    Status:        Pending
    Volume:
    Labels:        <none>
    Annotations:   <none>
    Finalizers:    [kubernetes.io/pvc-protection]
    Capacity:
    Access Modes:
    VolumeMode:    Filesystem
    Used By:       <none>
    Events:
      Type     Reason              Age               From                         Message
      ----     ------              ----              ----                         -------
      Warning  ProvisioningFailed  1s (x4 over 31s)  persistentvolume-controller  no volume plugin matched name: kubernetes.io/no-provisioner
    ```
  - Create a Pod
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
  - Status: Pod is in Pending State
    ```yaml
    $ k get pods,pvc
    NAME             READY   STATUS    RESTARTS   AGE
    pod/web-server   0/1     Pending   0          3s
    
    NAME                                   STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
    persistentvolumeclaim/web-assets-pvc   Pending                                      local-link     <unset>                 10m
    
    user1@LAPTOP-NV3Q15EM:~/cloud-joshi/storage$ k describe pod/web-server
    Name:             web-server
    Namespace:        default
    Priority:         0
    Service Account:  default
    Node:             <none>
    Labels:           run=web-server
    Annotations:      <none>
    Status:           Pending
    IP:
    IPs:              <none>
    Containers:
      web-server:
        Image:        nginx
        Port:         <none>
        Host Port:    <none>
        Environment:  <none>
        Mounts:
          /usr/share/nginx/html from local-pvc (rw)
          /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-shlmc (ro)
    Conditions:
      Type           Status
      PodScheduled   False
    Volumes:
      local-pvc:
        Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
        ClaimName:  web-assets-pvc
        ReadOnly:   false
      kube-api-access-shlmc:
        Type:                    Projected (a volume that contains injected data from multiple sources)
        TokenExpirationSeconds:  3607
        ConfigMapName:           kube-root-ca.crt
        Optional:                false
        DownwardAPI:             true
    QoS Class:                   BestEffort
    Node-Selectors:              <none>
    Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
    Events:
      Type     Reason            Age   From               Message
      ----     ------            ----  ----               -------
      Warning  FailedScheduling  8s    default-scheduler  0/3 nodes are available: pod has unbound immediate PersistentVolumeClaims. not found
        
    ```
  - Solution:
    - Because your StorageClass uses provisioner: kubernetes.io/no-provisioner, the Kubernetes control plane is "handcuffed"—it has been explicitly told not to create any volumes automatically. The Pod and PVC are pending because they are waiting for a physical volume that doesn't exist yet.
    - The Solution: Manually Create the PV
To resolve the FailedScheduling error, you must act as the Administrator and provide a PersistentVolume (PV) that matches the PVC’s requirements and the StorageClass's name.     
</details>

<details>
  <summary>Create a storageclass with Immediate volumebindingmode, and attach pvc to it</summary>

  - Create a storage class
    ```yaml
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      annotations:
        storageclass.kubernetes.io/is-default-class: "true"
      name: standard
    provisioner: rancher.io/local-path
    reclaimPolicy: Delete
    volumeBindingMode: Immediate  
    ```
    
  - Create a PVC
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
      storageClassName: "standard"
    
    ```

  - PVC status
    ```yaml
    $ k describe pvc
    Name:          web-assets-pvc
    Namespace:     default
    StorageClass:  standard
    Status:        Pending
    Volume:
    Labels:        <none>
    Annotations:   volume.beta.kubernetes.io/storage-provisioner: rancher.io/local-path
                   volume.kubernetes.io/storage-provisioner: rancher.io/local-path
    Finalizers:    [kubernetes.io/pvc-protection]
    Capacity:
    Access Modes:
    VolumeMode:    Filesystem
    Used By:       <none>
    Events:
      Type     Reason                Age                  From                                                                                                Message
      ----     ------                ----                 ----                                                                                                -------
      Normal   Provisioning          37s (x6 over 4m14s)  rancher.io/local-path_local-path-provisioner-5c4cdb564f-7nq48_9d52d8f7-96b7-4ad9-9d1f-fb53a0fff678  External provisioner is provisioning volume for claim "default/web-assets-pvc"
      Warning  ProvisioningFailed    37s (x6 over 4m14s)  rancher.io/local-path_local-path-provisioner-5c4cdb564f-7nq48_9d52d8f7-96b7-4ad9-9d1f-fb53a0fff678  failed to provision volume with StorageClass "standard": configuration error, no node was specified
      Normal   ExternalProvisioning  9s (x18 over 4m14s)  persistentvolume-controller                                                                         Waiting for a volume to be created either by the external provisioner 'rancher.io/local-path' or manually by the system administrator. If volume creation is delayed, please verify that the provisioner is running and correctly registered.        
    ```
</details>

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

<details>
  <summary>Warning  ProvisioningFailed  7s    persistentvolume-controller  no volume plugin matched name: kubernetes.io/no-provisioner</summary>

  - The controller sees your PVC requesting that StorageClass. It looks for a "plugin" (a piece of code) that knows how to create a volume for no-provisioner. Since there is no such plugin (by design), it throws a Warning
  - To move past this warning and get your Pod running, you must fulfill the Static Provisioning contract:
    - The Admin Action: You must manually create a PersistentVolume (PV) object.
    - The Handshake: That PV must have the storageClassName: local-link to match your PVC.
    - The Result: Once the PV exists, the controller stops trying to "provision" (which was failing) and instead performs a "bind" (which will succeed).
</details>

<details>
  <summary> Warning  ProvisioningFailed    12s               rancher.io/local-path_local-path-provisioner-5c4cdb564f-7nq48_9d52d8f7-96b7-4ad9-9d1f-fb53a0fff678  failed to provision volume with StorageClass "standard": configuration error, no node was specified</summary>

  - The error failed to provision volume... no node was specified is happening because:
    - Immediate Binding: Your StorageClass tells Kubernetes to create the volume right now, before a Pod even exists.
    - Local Path Constraint: Unlike a network drive (like AWS EBS or NFS) that can hang out in the cloud until needed, a local path volume must live on a specific node's hard drive.
    - The Conflict: Since there is no Pod, the provisioner has no idea which node to pick. It's looking for a node hint that isn't there.
  - The Fix: Update your StorageClass
    - You need to change the volumeBindingMode. This tells the provisioner: "Wait until a Pod is scheduled to a node, then create the storage on that specific node."

</details>

<details>
  <summary>Warning  FailedMount  74s (x10 over 5m15s)  kubelet  MountVolume.NodeAffinity check failed for volume "node-locked-pv" : no matching NodeSelectorTerm</summary>

  - if you see a Pod stuck in Pending and kubectl describe says "volume node affinity conflict," it means:
    - The PV is restricted to certain nodes/zones.
    - The Pod was scheduled (or forced) to a node that cannot "see" that storage.
</details>
