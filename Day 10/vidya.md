# Deployment Strategies

## Extra questions

<details>
  <summary>what is 25% of 1 replica</summary>

  When you specify maxUnavailable as a percentage, Kubernetes calculates the absolute number and rounds down to the nearest integer.
  
  Calculation: 1 replicas * 25% = 0.25%
  
  Rounding: floor(0.25) = 0

  - Because the result is 0, Kubernetes is not allowed to take your single pod offline to start the update. To perform a rolling update, it must follow this sequence:
  - Surge First: It will first spin up a new pod (the new version).
  - Readiness: It will wait until that new pod passes its readiness probes.
  - Terminate Old: Only after the new pod is "Ready" will it terminate your old pod.

  The default maxSurge is also 25%, which rounds up to 1.
  
</details>

<details>
  <summary>maxSurge: 0% , maxUnavailable: 25% , replicas=1</summary>

  -  While the math ($floor(0.25) = 0$) says it should be unavailable, Kubernetes has a built-in rule: If the calculation for maxUnavailable results in 0, but maxSurge is also 0, Kubernetes automatically treats maxUnavailable as 1.
  -  Since you had maxSurge: 0%, Kubernetes was forbidden from creating a second pod first. To satisfy the need to update, it performed a Recreate-like behavior:
     - It saw that it had to make progress.
     - It killed your existing pod (since maxUnavailable was boosted to 1).
     - It immediately started the new pod.

  - Even though it worked, this setup guarantees downtime. Between the moment the old pod was deleted and the new one became Running, there was no pod serving traffic.
    
</details>

<details>
  <summary>When you have both maxUnavailable: 25% and maxSurge: 25% set for 1 replica</summary>

  - Kubernetes interprets them as 0 and 1 respectively
  - so one new pod  will be created first only then old  pod is deleted
</details>

<details>
  <summary>what happens if both maxUnavailable and maxSurge is specified as 0</summary>

  - It wont allow
    - spec.strategy.rollingUpdate.maxUnavailable: Invalid value: "0%": may not be 0 when `maxSurge` is 0
    - spec.strategy.rollingUpdate.maxUnavailable: Invalid value: 0: may not be 0 when `maxSurge` is 0
    
</details>

<details>
  <summary>maxSurge will create how many pods for 1 replica</summary>

  - On a deployment with only 1 replica, 25%, 50%, and 100% all behave exactly the same because they all round up to 1.

| Desired Replicas | 25% Calculation | Absolute Surge (Round Up) | Max Pods During Update | Update Behavior |
| :--- | :--- | :--- | :--- | :--- |
| **1** | 0.25 | **1** | 2 | **Zero Downtime:** 1 new pod starts before the 1 old pod is removed. |
| **4** | 1.00 | **1** | 5 | **Safe/Slow:** Moves 1 pod at a time. |
| **8** | 2.00 | **2** | 10 | **Balanced:** Moves 2 pods at a time. |
| **10 | 2.50 | **3** | 13 | **Balanced:** Rounds up from 2.5 to 3. |
| **100** | 25.00 | **25** | 125 | **Fast:** Moves 25 pods at a time; requires high cluster capacity. |
</details>

<details>
  <summary>When to use Recreate rolling strategy </summary>

  - You might think "RollingUpdate" is always better because it has no downtime, but there are specific technical constraints where you must use strategy: type: Recreate.
  - Single-Access Storage (RWO Volumes): If your pod uses a Persistent Volume with ReadWriteOncePod access, only one pod can attach to the disk at a time. A Rolling Update will fail because the "New" pod can't mount the disk while the "Old" pod is still using it.
  - Database Migrations: If your new application version performs a destructive schema change (e.g., deleting a column) that the old version doesn't understand, running both versions simultaneously will cause the old pods to crash or corrupt data.

  - The Workflow of Recreate:
    - Terminates all Pods in Version A.
    - Waits for them to be fully removed (releasing locks/volumes).
    - Starts all Pods for Version B.
    - Result: Short downtime, but 100% data/state consistency.
    
  | Feature | RollingUpdate | Recreate |
  | :--- | :--- | :--- |
  | **Downtime** | Zero (if configured correctly) | **Guaranteed** downtime |
  | **Data Safety** | Risk of version mismatch | Highest safety for stateful apps |
  | **Speed** | Depends on readiness probes | Very fast (kills everything at once) |

</details>

## Scenarios

<details>
  <summary>Zero-Downtime with 1 Replica</summary>

  ```yaml
  $ k create deploy nginx --image nginx:1.16.0 --replicas=1
  deployment.apps/nginx created
  ```

  ```yaml
  # Edit the deployment to change the maxUnavailable from 25% to 0
  $ k get deploy nginx  -oyaml |grep -A3 strategy
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0
  ```

  ```yaml
  $ k set image deploy/nginx nginx=nginx:wrong
  deployment.apps/nginx image updated

  $ k get pods -w
  NAME                     READY   STATUS             RESTARTS   AGE
  nginx-64bc5c74c6-ffkxg   1/1     Running            0          82s
  nginx-69754ff47c-zrns6   0/1     ImagePullBackOff   0          44s
  ```
  
</details>

<details>
  <summary> maxUnavailable:25% maxSurge:25% replicas:4</summary>

  Calculation for your 4 replicas:
    - Desired Replicas: 4
    - MaxSurge (25% of 4): 1
    - Total allowed pods during update: 4 + 1 = 5

  Surge: Kubernetes wants to be fast, so it creates one new pod (nginx-69754ff47c-2cjpc) because maxSurge allows 1 extra. Total is now 5.
  Availability: Since your maxUnavailable is also 25% (1 pod), Kubernetes terminated one old pod to make room for a second new pod.

  Result: * 3 "Old" pods are still Running.
  - 2 "New" pods are stuck in ImagePullBackOff.
  - Total: $3 + 2 = 5$.

  - The rollout is now stalled. Kubernetes will not kill any more "Old" pods because it needs at least 3 healthy ones to satisfy the "max 1 unavailable" rule, and it won't create any more "New" pods because it has already hit the "max 5 total" limit.
  - What if I have only 1 replica? 25% of 1 is 0.25!" Kubernetes always rounds up for maxSurge and rounds down for maxUnavailable
  - Desired Replicas	maxSurge (25%)	maxUnavailable (25%)	Max Pods during Update
  - 1	                1	              0	                    2
  - 4	                1	              1	                    5
  - 10	              3	              2	                    13

  ```yaml
  $ k create deployment nginx --image nginx --replicas=4
  deployment.apps/nginx created

  $ k get pods -w
  NAME                     READY   STATUS    RESTARTS   AGE
  nginx-64bc5c74c6-b2thf   1/1     Running   0          4s
  nginx-64bc5c74c6-d262h   1/1     Running   0          4s
  nginx-64bc5c74c6-d8hcb   1/1     Running   0          4s
  nginx-64bc5c74c6-ztk74   1/1     Running   0          4s

  $ k set image deploy/nginx nginx=nginx:wrong
  deployment.apps/nginx image updated

  $ k get pods
  NAME                     READY   STATUS             RESTARTS   AGE
  nginx-64bc5c74c6-b2thf   1/1     Running            0          4m10s
  nginx-64bc5c74c6-d262h   1/1     Running            0          4m10s
  nginx-64bc5c74c6-d8hcb   1/1     Running            0          4m10s
  nginx-69754ff47c-2cjpc   0/1     ImagePullBackOff   0          4m
  nginx-69754ff47c-cmvhb   0/1     ImagePullBackOff   0          4m
  
  ```
</details>




## Errors
