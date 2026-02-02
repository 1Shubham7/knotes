#### What is route

same as HTTPRoute, just a way to say traffic for "abc.com/path" goes to this pod. a route defines how incoming traffic should be handled. It specifies things like:

- What host/domain the request is for (e.g., parent1.com)
- What path the request has (e.g., /anything/team1/foo)
- What service to send it to
- What transformations to apply (like adding/removing headers)


#### What is route delegation

Route Delegation is a feature where you can split route definitions across multiple resources instead of putting everything in one place. Basically have parent route send traffic to child route. 

- Route Delegation is Gateway API feature, not exclusive to kgateway.

```
# PARENT ROUTE: Handles the hostname level
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: parent-route
  namespace: default
spec:
  parentRefs:
  - name: my-gateway
  hostnames:
  - "api.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: child-users-route
      kind: HTTPRoute
    - name: child-products-route
      kind: HTTPRoute

---

# CHILD ROUTE 1: Handles /api/users paths
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: child-users-route
  namespace: default
spec:
  parentRefs:
  - name: parent-route
    kind: HTTPRoute
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /users
    backendRefs:
    - name: user-service
      port: 8080

---

apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: child-products-route
  namespace: default
spec:
  # This also references the parent route
  parentRefs:
  - name: parent-route
    kind: HTTPRoute
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /products
    backendRefs:
    - name: product-service
      port: 8080
```

```
# PARENT POLICY (applied at hostname level)
apiVersion: networking.kgateway.io/v1
kind: TrafficPolicy
metadata:
  name: parent-policy
spec:
  targetRefs:
  - kind: HTTPRoute
    name: parent-route
  http:
    response:
      addHeaders:
      - name: "origin"
        value: "parent1"
      - name: "environment"
        value: "production"

---

# CHILD POLICY (applied at path level)
apiVersion: networking.kgateway.io/v1
kind: TrafficPolicy
metadata:
  name: child-policy
spec:
  targetRefs:
  - kind: HTTPRoute
    name: child-route
  http:
    response:
      addHeaders:
      - name: "origin"
        value: "svc1"
      - name: "service"
        value: "user-service"

---
```
### What is Shallow merge vs Deep merge

It is basically a field in manifests  `mergeStrategy: DeepMerge`

Shallow means if the child has something, it COMPLETELY REPLACES the parent's version. Deep merge on the other hand checks each field, and choses the child one if they have conflict, otherwise, it chooses everything from parent.

```
# PARENT POLICY with DeepMerge strategy
apiVersion: networking.kgateway.io/v1
kind: TrafficPolicy
metadata:
  name: parent-policy
  namespace: default
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: parent-route
  # THIS is where you specify the merge strategy!
  mergeStrategy: DeepMerge    # ← MERGE STRATEGY SPECIFIED HERE
  http:
    response:
      addHeaders:
      - name: "origin"
        value: "parent1"
      - name: "environment"
        value: "production"

---

# CHILD POLICY with ShallowMerge strategy
apiVersion: networking.kgateway.io/v1
kind: TrafficPolicy
metadata:
  name: child-policy
  namespace: default
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: child-route
  # THIS is where you specify the merge strategy!
  mergeStrategy: ShallowMerge   # ← MERGE STRATEGY SPECIFIED HERE
  http:
    response:
      addHeaders:
      - name: "origin"
        value: "svc1"
      - name: "service"
        value: "user-service"
```

ShallowMerge:
- origin: svc1              (from child)
- service: user-service     (from child)


DeepMerge:
- origin: svc1              (child's value overrides parent's)
- environment: production   (from parent - no conflict, so included)

#### RustFormation

RustFormation is a module that uses native Envoy per-route config Kgateway for transforming traffic. it is a transformation engine written in Rust. It's more powerful and works at a more granular level (can merge at the "set", "add", and "remove" level for headers) . This is the one that  exposes the bug we are seeing.















