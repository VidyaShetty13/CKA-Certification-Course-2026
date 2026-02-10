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
  <summary>We are deploying a application in a multi-node cloud environment. Why would we choose WaitForFirstConsumer over Immediate as the volume binding mode in our StorageClass?</summary>

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


## storage class scenarios

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
