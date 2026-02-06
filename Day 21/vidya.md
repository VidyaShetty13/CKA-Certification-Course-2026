# Scenarios

1. Maximum num of initContainers and containers that can be present in a pod?
   - In Kubernetes, there is no hard-coded "maximum number" for the number of containers or init containers in a Pod.
   - However, you are limited by the Resource Capacity of your nodes and the etcd Object Size Limit.
   - While the API won't stop you from adding 50 containers, several factors will act as a "ceiling":
     - Node Resources (CPU/RAM): Every container has an overhead. If you have 100 containers in one Pod, the Kubelet on that node has to manage 100 separate processes, health checks, and log streams. You will likely hit a "Resource Exhaustion" wall long before you hit an API limit.
     - etcd Limit: Kubernetes stores Pod definitions in etcd. The default maximum size for any single object in etcd is 1.5 MB. A Pod YAML with hundreds of containers, each with its own environment variables, volume mounts, and probes, could eventually exceed this size.
     - IP Exhaustion: All containers in a Pod share a single IP address. However, they must use different ports. If you have too many containers, managing port conflicts becomes a nightmare
    
     - Technically, there is no fixed limit in the Kubernetes source code. However, practically, the limit is defined by the 1.5MB etcd object size limit and the resource capacity of the node where the Pod is scheduled. In practice, most Pods rarely exceed 5 or 6 total containers to keep the blast radius small and troubleshooting simple
   
2. Can we pecify resources for initContainers?
   Yes

3. Do init containers support pod lifecyle or health probes?
   init containers do not support lifecycle, livenessProbe, readinessProbe, or startupProbe whereas sidecar containers support all these probes to control their lifecycle.

4. So restartPolicy of a pod is also applicable to initContainers?
   - When you apply this, Kubernetes will enter a loop. Here is the lifecycle you will observe:
     - Pod Status: Init:CrashLoopBackOff or Init:Error.
     - App Containers: They will never start. If you run kubectl get pods, you will see 0/1 containers ready.
     - Restarts: Kubernetes will restart the initContainer over and over based on your Pod's restartPolicy (which is Always by default in a Deployment).
   - Dependency: Main containers are blocked until all init containers exit with code 0.
     
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: init-demo-2
      labels:
        app: main-app
    spec:
      initContainers:
      - name: check-api
        image: curlimages/curl:latest
        command:
          - sh
          - -c
          - |
            echo 'Checking external API availability...'
            curl -s https://kubernetes-wrong.io > /dev/null || { echo 'API CHECK FAILED'; exit 1; }
            echo 'External API is accessible, proceeding to next init container!'
    
      - name: check-svc
        image: curlimages/curl:latest
        command:
          - sh
          - -c
          - |
            echo 'Checking main-app Service availability...'
            until nslookup main-app-svc.default.svc.cluster.local; do
              echo 'Waiting for Service DNS resolution...'
              sleep 5
            done
            echo 'Service is reachable, proceeding to main-app container!'
    
      containers:
      - name: main-app
        image: nginx:latest
    ```
    
    ```yaml
    $ k logs init-demo-2 -c check-api -f
    Checking external API availability...
    API CHECK FAILED
    ```
    
    ```yaml
    $ k get pods
    NAME          READY   STATUS                  RESTARTS      AGE
    init-demo-2   0/1     Init:CrashLoopBackOff   4 (46s ago)   2m25s
    
    ```
5. If the initcontainers are succeeded, but the main container is in crashloop state, will this restartPolicy: Always will restart the init containers?
   - The Init Phase: Kubernetes runs check-api and check-svc. Since they both succeeded (exited with code 0), Kubernetes marks the Init Phase as "Completed."      The Success Record: Once an Init Container succeeds, Kubernetes does not run it again for the life of that Pod, even if the main container crashes. It "saves its progress."
   - The Crash Loop: Your main container runs kill 1, which causes it to crash. Kubernetes looks at the Pod's restartPolicy (default: Always) and says, "I need to restart the main container."
   - Why the Init Containers aren't re-running
     - If Kubernetes re-ran Init Containers every time a main container crashed, it would be incredibly inefficient. Imagine an Init Container that does a massive database migration taking 10 minutes—you wouldn't want that running every time your Nginx process restarts!
     - Key Rule: Init Containers only re-run if the entire Pod is restarted (e.g., if the Node reboots or the Pod is deleted and recreated).
     - The RESTARTS Column: This column in kubectl get pods actually shows the aggregate sum of restarts for all containers in the Pod, but in your case, it’s only the main-app container contributing to that number because the Init Containers are already "Completed."

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: init-demo-2
      labels:
        app: main-app
    spec:
      initContainers:
      - name: check-api
        image: curlimages/curl:latest
        command:
          - sh
          - -c
          - |
            echo 'Checking external API availability...'
            curl -s https://kubernetes.io > /dev/null || { echo 'API CHECK FAILED'; exit 1; }
            echo 'External API is accessible, proceeding to next init container!'
    
      - name: check-svc
        image: curlimages/curl:latest
        command:
          - sh
          - -c
          - |
            echo 'Checking main-app Service availability...'
            until nslookup main-app-svc.default.svc.cluster.local; do
              echo 'Waiting for Service DNS resolution...'
              sleep 5
            done
            echo 'Service is reachable, proceeding to main-app container!'
    
      containers:
      - name: main-app
        image: nginx:latest
        command:
          - sh
          - -c
          - |
            kill 1
    ---
    $ k get pods -w
    NAME          READY   STATUS     RESTARTS   AGE
    init-demo-2   0/1     Init:0/2   0          3s
    init-demo-2   0/1     Init:1/2   0          4s
    init-demo-2   0/1     PodInitializing   0          6s
    init-demo-2   0/1     Completed         0          8s
    init-demo-2   0/1     Completed         1 (2s ago)   10s
    init-demo-2   0/1     CrashLoopBackOff   1 (2s ago)   11s
    init-demo-2   0/1     Completed          2 (15s ago)   24s
    init-demo-2   0/1     CrashLoopBackOff   2 (15s ago)   38s
    init-demo-2   0/1     Completed          3 (29s ago)   52s
    init-demo-2   0/1     CrashLoopBackOff   3 (14s ago)   65s
    init-demo-2   0/1     Completed          4 (56s ago)   107s
    init-demo-2   0/1     CrashLoopBackOff   4 (15s ago)   2m2s
    ```

6. Explain :- Each init container must exit successfully before the next container starts. If a container fails to start due to the runtime or exits with failure, it is retried according to the Pod restartPolicy. However, if the Pod restartPolicy is set to Always, the init containers use restartPolicy OnFailure.
   - 1. The Sequential Requirement
        "Each init container must exit successfully before the next container starts."
        Kubernetes treats Init Containers as a strict checklist. If you have Init-A, Init-B, and Main-App, the flow is a "Pass/Fail" gate.
        - If Init-A fails, the entire process stops there.
        - Init-B will never even attempt to start until Init-A returns an Exit Code 0.
   - 2. The Retrying Logic
        "If a container fails to start due to the runtime or exits with failure, it is retried according to the Pod restartPolicy."
        - If your Init Container fails (e.g., your script has a bug or the network is down), Kubernetes doesn't just give up immediately. It looks at the restartPolicy you defined for the whole Pod:
        - OnFailure: It will restart the Init Container.
        - Never: It will mark the Pod as Failed and stop.
   - 3. The "Always" vs. "OnFailure" Override (The most important part)
        "However, if the Pod restartPolicy is set to Always, the init containers use restartPolicy OnFailure."
        - This is a clever piece of built-in logic. Usually, a Deployment has a restartPolicy: Always because we want our Main App to run forever.
        - But think about an Init Container: Its job is to finish and exit. If it "always" restarted, it would run in an infinite loop, and your Main App would never start.
        - The Kubernetes Logic: If you set the Pod to Always, Kubernetes "downgrades" that policy to OnFailure specifically for the Init Containers.
        - What this means: * If the Init Container succeeds (Exit 0): It stays finished. The pod moves to the next stage.
        - If the Init Container fails (Exit 1): It restarts so it can try to succeed again.
