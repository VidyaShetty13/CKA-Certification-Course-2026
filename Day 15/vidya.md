# Static Pod Scenarios

<details>
  <summary>1. how kubernetes controlplane components are installed </summary>

  - using static pods
</details>

<details>
  <summary>2. Is static pod to be placed only in controlplane?</summary>

  - No, static pod can be placed even in worker nodes, its not mandatory to be placed only in controlplane node
</details>

<details>
  <summary>3. how the static pods are seen when ran kubectl commands, when in general static pods are not managed by the api-server, does api-server store staticpod details in etcd?</summary>

  - kubelet creates a Mirror Pod on the API server. This Mirror Pod is a "read-only" representation of what the Kubelet is doing locally. the API server accepts this pod and stores it in its database (etcd).
  - annotations of the pod:- kubernetes.io/config.mirror: 23963e82431aa75cc7b33ba981c371f0

</details>

<details>
  <summary>4. can we edit the static pods directly using kubectl edit command </summary>

  - yes, we can edit, but kubelet will revert the change back to manifest file 
</details>

<details>
  <summary>5. who says that the static pods has to be placedin /etc/kubernetes/manifest and not in some other folder?</summary>

  - While /etc/kubernetes/manifests is the industry standard, it is not hardcoded into Kubernetes. You can change it to /my/secret/pods or any other folder you like.
  - login to the node and get the congifile path
    ```yaml
    root@kind-worker:/# ps -ef |grep kubelet
    root         200       1  2 10:23 ?        00:04:26 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --node-ip=172.19.0.4 --node-labels= --pod-infra-container-image=registry.k8s.io/pause:3.10.1 --provider-id=kind://docker/kind/kind-worker --runtime-cgroups=/system.slice/containerd.service
    ```
  - The path is defined in the Kubelet Configuration file (usually found at /var/lib/kubelet/config.yaml).
</details>

<details>
  <summary>6. What happens if I put a non-Pod YAML (like a Service or Deployment) in that folder?</summary>

  - The Kubelet will try to run it, fail, and log an error. The Kubelet only understands Pod specs. It doesn't know what a "Deployment" is—that's the job of the Control Plane (which static pods are designed to bypass).
</details>


<details>
  <summary>7. If I use kubectl delete pod on a static pod, what happens? Will it ever go away?</summary>

  - No. If you delete the Mirror Pod using kubectl, it will briefly disappear from the kubectl get pods output, but the Kubelet on the node will immediately recreate the mirror because the source file still exists in /etc/kubernetes/manifests
  - To truly delete a static pod, you must physically remove the YAML file from the node's manifest folder. The Kubelet watches that folder using inotify and will terminate the container once the file is gone
</details>

<details>
  <summary>8. How can you tell if a pod is a Static Pod just by looking at kubectl get pods?</summary>
  - The Name: Static pods almost always have the node name appended to them (e.g., kube-apiserver-kind-control-plane).
  - If you run kubectl get pod <name> -o yaml, look at the ownerReferences. A regular pod is owned by a ReplicaSet or DaemonSet. A static pod is owned by the Node itself.
  
</details>

<details>
  <summary>9. Can a static pod use a Secret or a ConfigMap mounted as a volume?</summary>
  - Static pods are managed entirely by the Kubelet without talking to the API server. Since Secrets and ConfigMaps live in the API server's database (etcd), the Kubelet has no way to fetch them before starting the static pod.
  - If a static pod needs configuration, you must place those config files directly on the node's host filesystem and use a hostPath volume.
</details>

<details>
  <summary>10. Does the kube-scheduler play any role in where a static pod runs?</summary>

  - No
  - Static pods are bound to the specific node where their file resides. In fact, the kube-scheduler itself is usually a static pod! It would be a "chicken and egg" problem if the scheduler had to schedule itself.
  - 
</details>

<details>
  <summary>11. A static pod is failing to start. kubectl logs shows nothing because the pod isn't even 'Running'. How do you debug it?</summary>

  - Since the Kubelet is the parent process, you must go to the node's OS-level logs.
  - 1. SSH into the node. 2. Run journalctl -u kubelet -f. 3. Look for errors related to "Pod manifest" or "ImagePullBackOff".
    2. Check if the YAML syntax is valid. If there's a typo in the manifest, the Kubelet will simply ignore the file and won't even try to create a Mirror Pod.
</details>
