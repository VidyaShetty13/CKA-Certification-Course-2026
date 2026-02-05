# Scenarios with Service

## Questions

<details>
  <summary>1. Can the service be created without selectors</summary>

yes. Without selectors it will be `ExternalName` service type
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
  - 
</details>

<details>
  <summary>5. How to set the loadbakancer service without creating clusterIP or nodePort</summary>

  - to create loadbalancer without nodePort
    ```yaml
    You can optionally disable node port allocation for a Service of type: LoadBalancer, by setting the field spec.allocateLoadBalancerNodePorts to false
    ```
  - 
</details>

<details>
  <summary>6. How to map a Service directly to a specific Pod IP address</summary>
  - Use headless service by explicitly specifying "None" for the cluster IP address (.spec.clusterIP)
  - For headless Services, a cluster IP is not allocated, kube-proxy does not handle these Services, and there is no load balancing or proxying done by the platform for them.
  - To define a headless Service, you make a Service with .spec.type set to ClusterIP (which is also the default for type), and you additionally set .spec.clusterIP to None.
</details>












