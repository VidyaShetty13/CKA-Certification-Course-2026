# RBAC

## Extra Questions

<details>
  <summary>Difference get and list verbs</summary>

  - get :- Retrieve details of a specific resource (e.g., kubectl get pod nginx -o yaml)
  - list :- List resources of a certain type (e.g., kubectl get pods)
</details>



## Scenarios

<details>
  <summary>Difference between get and list verbs in RBAC</summary>

  - Created 1 users named seema
    ```yaml
    $ k config get-contexts
    CURRENT   NAME            CLUSTER     AUTHINFO    NAMESPACE
    *          kind-kind       kind-kind   kind-kind   default
             seema-context   kind-kind   seema
    
    $ k config get-users
    NAME
    kind-kind
    seema
    ```
    
  - As admin:- Created a role which allows seems to list the pods in default namespace
    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      namespace: default
      name: pod-lister
    rules:
    - apiGroups: [""]
      resources: ["pods"]
      verbs: ["list"] # ONLY list
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: seema-list-binding
      namespace: default
    subjects:
    - kind: User
      name: seema
      apiGroup: rbac.authorization.k8s.io
    roleRef:
      kind: Role
      name: pod-lister
      apiGroup: rbac.authorization.k8s.io
    ```
    
  - Now switch the context to seema and try to list the pods
    ```yaml
    $ k config use-context seema-context
    Switched to context "seema-context".
    
    $ k config get-contexts
    CURRENT   NAME            CLUSTER     AUTHINFO    NAMESPACE
              kind-kind       kind-kind   kind-kind   default
    *         seema-context   kind-kind   seema
    ```
    
  - Now as seema try to get the pod details
    ```yaml
    # LIST
    $ k get pods -n default
    NAME    READY   STATUS    RESTARTS   AGE
    nginx   1/1     Running   0          5m52s

    # GET
    $ k get pods -n default nginx
    Error from server (Forbidden): pods "nginx" is forbidden: User "seema" cannot get resource "pods" in API group "" in the namespace "default"    
    ```

    ```yaml
    $ k config use-context kind-kind
    Switched to context "kind-kind".
    
    $ kubectl auth can-i get pods --as=seema -n default
    no
    ```
    
</details>

<details>
  <summary>Try creating a role that allows access to nodes</summary>

  ```yaml
  $ cat role.yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    namespace: default
    name: pod-lister
  rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["list"] # ONLY list
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get"]
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: seema-list-binding
    namespace: default
  subjects:
  - kind: User
    name: seema
    apiGroup: rbac.authorization.k8s.io
  roleRef:
    kind: Role
    name: pod-lister
    apiGroup: rbac.authorization.k8s.io
  ```

  ```yaml
  $ k get pods
  NAME    READY   STATUS    RESTARTS   AGE
  nginx   1/1     Running   0          10m
  
  $ k get nodes kind-control-plane
  Error from server (Forbidden): nodes "kind-control-plane" is forbidden: User "seema" cannot get resource "nodes" in API group "" at the cluster scope
  ```
  - Results
    - A role cannot have cluster resource
      
</details>

<details>
  <summary> Create a clusterrole that allows seema user to get and list the node details</summary>

  - create a clusterrole and clusterrolebinding
    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: node-access
    rules:
    - apiGroups:
      - ""
      resources:
      - nodes
      verbs:
      - get
      - list
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: node-binding
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: node-access
    subjects:
    - apiGroup: rbac.authorization.k8s.io
      kind: User
      name: seema
    
    ```
  - switch to seema context
    ```yaml
    $ k config use-context seema-context
    Switched to context "seema-context".
    ```
    
  - as seema try to get the node details
    ```yaml
    $ k get nodes
    NAME                 STATUS   ROLES           AGE   VERSION
    kind-control-plane   Ready    control-plane   16d   v1.34.3
    kind-worker          Ready    <none>          16d   v1.34.3
    kind-worker2         Ready    <none>          16d   v1.34.3

    $ k get nodes kind-control-plane
    NAME                 STATUS   ROLES           AGE   VERSION
    kind-control-plane   Ready    control-plane   16d   v1.34.3
    ```
   
</details>


<details>
  <summary> Create a clusterrole that allows seema user to get and list the deployment(from default namespace) and node details </summary>

  - create a clusterrole and rolebinding on default namespace
    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: node-access
    rules:
    - apiGroups:
      - ""
      resources:
      - nodes
      verbs:
      - get
      - list
    - apiGroups:
      - "apps"
      resources:
      - deployments
      verbs:
      - get
      - list
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: node-binding
      namespace: default
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: node-access
    subjects:
    - apiGroup: rbac.authorization.k8s.io
      kind: User
      name: seema
        
  - switch to seema context
    ```yaml
    $ k config use-context seema-context
    Switched to context "seema-context".
    ```
  - as seema try to get node and deployment details
    - deployment details is sucecssful
      ```yaml
      $ k get deploy
      NAME    READY   UP-TO-DATE   AVAILABLE   AGE
      nginx   1/1     1            1           17m
      
      $ k get deploy nginx
      NAME    READY   UP-TO-DATE   AVAILABLE   AGE
      nginx   1/1     1            1           17m
      ```
      
    - node details failed
      ```yaml
      $ k get nodes
      Error from server (Forbidden): nodes is forbidden: User "seema" cannot list resource "nodes" in API group "" at the cluster scope

      ```
    - failure because
      - Because you used a RoleBinding, the API server looks at your request for nodes and says: "Seema has 'node-access' inside the 'default' namespace... but there are no nodes inside the 'default' namespace. Request Denied."
      - You need to delete the local RoleBinding and create a ClusterRoleBinding. This gives Seema the power to see nodes (which are cluster-wide)
      - If a resource is Cluster-Scoped (like Nodes, PersistentVolumes, or Namespaces), a RoleBinding is practically useless because a RoleBinding only grants power within the "four walls" of a specific namespace.   
</details>


<details>
  <summary>Why the RoleBinding + ClusterRole combo exists</summary>

  - You might ask: "If I have a ClusterRole, why would I ever use a regular Role?"
    - It is used for Template Efficiency. Imagine you have 100 namespaces and you want a "Secret Reader" in each one.
    - Instead of creating 100 identical Roles, you create one ClusterRole called secret-reader.
    - You then create a RoleBinding in each namespace pointing to that one global ClusterRole.
    - This gives the user access to secrets only in the namespace where the binding exists.
</details>


