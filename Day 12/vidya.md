# Scenarios with Service

## Questions

<details>
  <summary>1. Can the service be created without selectors</summary>

When you create a Service with a selector, Kubernetes automatically creates and manages an Endpoints (or EndpointSlice) object for you. When you omit the selector, Kubernetes creates the Service but leaves it "empty"—it has no idea where to send traffic until you manually provide the destination

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx
  name: nginx
  namespace: default
spec:
  clusterIP: None
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80

---
$ k describe svc nginx
Name:                     nginx
Namespace:                default
Labels:                   app=nginx
Annotations:              <none>
Selector:                 <none>
Type:                     ClusterIP
IP Family Policy:         RequireDualStack
IP Families:              IPv4,IPv6
IP:                       None
IPs:                      None
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
Endpoints:                <none>
Session Affinity:         None
Internal Traffic Policy:  Cluster
Events:                   <none>

```
</details>

<details>
  <summary>2. How to point your Service to a Service in a different Namespace or on another cluster</summary>

by using `ExternalName` service type

**service created in test namespace**
```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2026-02-05T12:27:39Z"
  labels:
    app: test
  name: test
  namespace: test
  resourceVersion: "225578"
  uid: 127e183b-7c5f-4913-b9df-8132f73627af
spec:
  clusterIP: 10.96.44.154
  clusterIPs:
  - 10.96.44.154
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: test
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```

**External service created in default namespace to point to test namespace svc name**
```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: test
  name: nginx-svc
spec:
  type: ExternalName
  externalName: test.test.svc.cluster.local
```

**nslookup on nginx-svc**
```yaml
/ # nslookup nginx-svc
Server:         10.96.0.10
Address:        10.96.0.10:53

** server can't find nginx-svc.cluster.local: NXDOMAIN

** server can't find nginx-svc.cluster.local: NXDOMAIN

nginx-svc.default.svc.cluster.local     canonical name = test.test.svc.cluster.local
Name:   test.test.svc.cluster.local
Address: 10.96.44.154

** server can't find nginx-svc.svc.cluster.local: NXDOMAIN

** server can't find nginx-svc.svc.cluster.local: NXDOMAIN
```

</details>

<details>
  <summary>3. What is the maximum num of Endpoints can be present in a Endpointslices before kubernetes creates a new EndpointSlice</summary>

  100
</details>

<details>
  <summary>4. How to set the nodePort Service without creating clusterIP</summary>
  
  - Its not possible. NodePort requires ClusterIP
  
    ```yaml
    $ k create -f svc.yaml
    The Service "nginx" is invalid: spec.clusterIPs[0]: Invalid value: "None": may not be set to 'None' for NodePort services
  
    $ cat svc.yaml
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        app: nginx
      name: nginx
      namespace: default
    spec:
      clusterIP: None
      ports:
      - port: 80
        protocol: TCP
        targetPort: 80
      selector:
        app: nginx
      type: NodePort
    ```
</details>

<details>
  <summary>5. How to set the loadbalancer service without creating clusterIP or nodePort</summary>

  - to create loadbalancer without nodePort
    - You can optionally disable node port allocation for a Service of type: LoadBalancer, by setting the field spec.allocateLoadBalancerNodePorts to false
  - to create loadbalancer without ClusterIP
    - Most Cloud Providers (AWS, GCP, Azure) will fail to provision a Load Balancer if the clusterIP is set to None. The cloud controller manager usually expects a functional ClusterIP to build the forwarding rules.
</details>

<details>
  <summary>6. How to map a Service directly to a specific Pod IP address</summary>
  
  - Use headless service by explicitly specifying "None" for the cluster IP address (.spec.clusterIP)
  - For headless Services, a cluster IP is not allocated, kube-proxy does not handle these Services, and there is no load balancing or proxying done by the platform for them.
  - To define a headless Service, you make a Service with .spec.type set to ClusterIP (which is also the default for type), and you additionally set .spec.clusterIP to None.
</details>

<details>
  <summary>7. what is kubernetes svc present in default namespace, why do we need it?</summary>

  - The kubernetes service acts as the internal gateway for pods to talk to the API Server.
  - Example: If a Pod (like a monitoring agent or a CI/CD runner) needs to list other pods or update a status, it doesn't need to know the Control Plane's actual IP. It simply targets https://kubernetes.default.svc.
  - The "Why": It provides a stable, predictable DNS name and IP address that remains constant even if the Control Plane nodes are replaced or moved.
    ```yaml
    $ k describe svc kubernetes -n default
    Name:                     kubernetes
    Namespace:                default
    Labels:                   component=apiserver
                              provider=kubernetes
    Annotations:              <none>
    Selector:                 <none>
    Type:                     ClusterIP
    IP Family Policy:         SingleStack
    IP Families:              IPv4
    IP:                       10.96.0.1
    IPs:                      10.96.0.1
    Port:                     https  443/TCP
    TargetPort:               6443/TCP
    Endpoints:                172.19.0.2:6443
    Session Affinity:         None
    Internal Traffic Policy:  Cluster
    Events:                   <none>
    ---
    $ k get endpointslices kubernetes  -oyaml
    addressType: IPv4
    apiVersion: discovery.k8s.io/v1
    endpoints:
    - addresses:
      - 172.19.0.2
      conditions:
        ready: true
    kind: EndpointSlice
    metadata:
      creationTimestamp: "2026-01-27T10:39:33Z"
      generation: 5
      labels:
        kubernetes.io/service-name: kubernetes
      name: kubernetes
      namespace: default
      resourceVersion: "232228"
      uid: 5ba269bb-bd76-4dd0-8245-3ebb67954725
    ports:
    - name: https
      port: 6443
      protocol: TCP
    ---
    $ k get nodes -owide
    NAME                 STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION                     CONTAINER-RUNTIME
    kind-control-plane   Ready    control-plane   9d    v1.34.3   172.19.0.2    <none>        Debian GNU/Linux 12 (bookworm)   6.6.87.2-microsoft-standard-WSL2   containerd://2.2.0

    ```
    
    - Why is the kubernetes service not created in every newly created namespace?
      - To maintain a Single Point of Truth and save resources
        - Example: If you have 100 namespaces, creating 100 identical services would clutter the cluster and waste 100 ClusterIPs.
        - How it works: Kubernetes is designed so that the default namespace service is globally reachable. Because of the way Kubernetes DNS (CoreDNS) works, a pod in a namespace called production can still reach the API server by simply calling kubernetes.default.

    - What happens if I delete the kubernetes service from the default namespace?
      - It is a temporary disruption followed by an automatic recovery.
        - Immediate Effect: Any Pod currently trying to communicate with the API server via that Service IP will fail immediately (Connection Refused).
        - The "Self-Healing": The kube-apiserver process constantly monitors this service. If it's deleted, the controller manager will automatically recreate it within seconds.
        - The Catch: While the service name (kubernetes) will return, it might get a new ClusterIP. If you have legacy applications that hardcoded the old IP address instead of using the DNS name, those applications will stay broken until restarted.
    

</details>


<details>
  <summary>8. How to debug a issue with service to pod communication</summary>

  - The most common issue is a "selector mismatch," where the Service exists but isn't pointing to any Pods.
    - kubectl get endpoints <service-name>
  - Test Internal DNS Resolution
    - Check if the cluster's DNS (CoreDNS) is actually resolving the Service name to an IP.
    - command: kubectl run busybox --image=busybox:1.28 -it --rm -- nslookup <service-name>
    - What to look for: It should return the ClusterIP. If it fails, the issue is likely with the CoreDNS pods or the internal cluster network.
  - Bypass the Service (Direct Pod Access)
    - To determine if the issue is the Service or the Application, try to hit the Pod directly.
      - Command: kubectl get pods -o wide (to get Pod IPs) and then curl <pod-ip>:<port> from another pod.
      - What to look for: If you can reach the Pod IP but not the Service IP, the problem lies in kube-proxy or the iptables rules.
  - Inspect Kube-Proxy and Iptables
    - If the Endpoints are correct but traffic still fails, the "plumbing" on the Node might be broken.
    - Command: iptables -t nat -L KUBE-SERVICES -n -v | grep <service-name> (as you did earlier).
    - What to look for: Check the packet counters (pkts). If the counter isn't increasing when you send traffic, the request isn't even reaching the iptables chain.
  - Check Network Policies
    - Finally, check if a NetworkPolicy is explicitly blocking the traffic.
      - Command: kubectl get networkpolicy -A
      - What to look for: A policy might be in place that allows traffic only from specific namespaces or labels, effectively "ghosting" your connection.

  - What if the iptables rules are perfect but traffic still drops?
    - Kubernetes uses the Linux conntrack (connection tracking) table to remember where masqueraded packets should go. If this table fills up, the service will "drop" connections intermittently.
    - sysctl net.netfilter.nf_conntrack_count
    - Host Level: (Are conntrack tables full or kernel modules missing?)
  
</details>

<details>
  <summary>9. How to debug a issue with pod to service communication</summary>

  - The DNS Check
    - Most Pods talk to Services using names (e.g., my-svc), not IPs.
    - Action: Exec into the source Pod and try to resolve the name. kubectl exec -it <pod-name> -- nslookup <service-name>
    If it fails: The issue is with CoreDNS. Check if the CoreDNS pods are running or if there's a problem with the kube-dns service.
  - The Endpoints Check
    - A Service is just a "front door." If there are no Pods behind it, the traffic has nowhere to go.
    - Action: kubectl get endpoints <service-name>
    - What to look for: If the ENDPOINTS list is <none> or empty, the Service labels do not match the Pod labels. This is the #1 most common error.
  - The Data Plane Check
    - If the Endpoints exist but the connection times out, you need to check the Node's routing rules.
    - For iptables mode: Check if the KUBE-SVC chain exists for that service (as you did earlier). iptables -t nat -L KUBE-SERVICES -n -v | grep <service-ip>
    - For IPVS mode: Check the virtual server table. ipvsadm -Ln -t <service-ip>:<port>
    - The "Packet Test": Watch the counters. If you send a request and the pkts count in iptables or the Conns count in ipvsadm doesn't increase, the packet isn't even reaching the load-balancing layer.
  - The "TargetPort"
    - A common mistake is having a Service listening on port 80, but the Pod is actually listening on port 8080.
    - Action: Check the targetPort in the Service YAML and compare it to the containerPort in the Deployment YAML.
    - The Test: Try to curl the Pod IP directly from the source Pod. If that works but the Service IP fails, the issue is definitely the Service configuration (ports or selectors).
  - Network Policies
    - If all the "plumbing" looks perfect but traffic still drops, a NetworkPolicy might be acting as a firewall.
    - Action: kubectl get networkpolicy -A
    - Check: Is there an Ingress rule on the destination Pod that allows traffic from the source Pod's namespace or labels?

</details>

<details>
  <summary>10. What is the difference between iptables mode and IPVS mode in kube-proxy?</summary>

  - iptables mode: Uses sequential rules. As the cluster grows (thousands of services), it gets slower because every packet has to traverse a long list of rules.
  - IPVS mode: Uses a hash table and is much faster for large-scale clusters. It also supports better load-balancing algorithms like "Least Connections" (iptables only does "Random").

</details>
