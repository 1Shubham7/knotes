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

---

# ISSUE

## Test Setup
Problem is with TestPolicyMerging (its being skipped right now), this is how the test setup is:

- two websites (`parent1.com` and `parent2.com`) share the same backend teams (`team1` and `team2`)
- Each website has its own rules about whose policy should win when there's a conflict.

we are trying to do something like :

```
Parent Route (platform team owns):
  "If the host is parent1.com and path starts with /anything/*, delegate to child routes"

Child Route (team1 owns):
  "If path starts with /anything/team1, route to my-service-1"

Child Route (team2 owns):  
  "If path starts with /anything/team2, route to my-service-2"
```

These are the manifests:

- Parent 1

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: parent1
  namespace: infra
  annotations:
    kgateway.dev/inherited-policy-priority: DeepMergePreferChild  # we will prefer child
spec:
  hostnames:
  - parent1.com                        # only matches requests to this host
  parentRefs:
  - name: http-gateway                 # Attached to the Gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /anything               # Matches any path starting with /anything
    backendRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: svc1                       # Delegates to child route "svc1"
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: svc2                       # Delegates to child route "svc2"
```

what this does is - When someone sends a request to `parent1.com/anything/*`, this route catches it and says "I'm not going to handle this myself — let my child routes (`svc1` and `svc2`) handle the specific sub-paths."

- Parent 2:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: parent2
  namespace: infra
  annotations:
    kgateway.dev/inherited-policy-priority: DeepMergePreferParent  # we will prefer parent
spec:
  hostnames:
  - parent2.com
  parentRefs:
  - name: http-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /anything
    backendRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: svc1
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: svc2
```

- Chile 1

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: svc1
  namespace: infra
spec:
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: parent1                      #  it is a child of both parent 1 and 2
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: parent2
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /anything/team1         #  it will delegate /team1 to httpbin
    backendRefs:
    - name: httpbin
```

This route handles `/anything/team1` requests. 

same goes for child 2,  `svc2` it matchs `/anything/team2` 

- TrafficPolicy on `parent2`

```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: TrafficPolicy
metadata:
  name: tp-parent2
  namespace: infra
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: parent2
  transformation:
    response:
      set:
      - name: origin
        value: parent2                 # in response it sets "origin" header to "parent2"
```

This policy says "for any request handled by `parent2`, set a response header `origin: parent2`."
And IMP -  There is **no** TrafficPolicy on `parent1`.

- TrafficPolicy on `svc1`

```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: TrafficPolicy
metadata:
  name: tp-svc1
  namespace: infra
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: svc1           
  transformation:
    response:
      set:
      - name: origin
        value: svc1                    #  Set "origin" header to "svc1"
```

and same for svc2 , just set header `origin: svc2`.


### What happens:

Request: `curl -H "host: parent1.com" http://gateway:8080/anything/team1/foo`

1. Gateway receives req
2. Host is "parent1.com" -> matches parent1 route
3. parent1 delegates to svc1 and svc2
4. Path starts with "/anything/team1" -> matches svc1's rule
5. Traffic goes to httpbin service

Policy merge:
  - svc1 has: origin=svc1
  - parent1 has: nothing (no TrafficPolicy on parent1)
  - parent1 annotation: DeepMergePreferChild -> child wins
  - Result: origin=svc1

Request: `curl -H "host: parent2.com" http://gateway:8080/anything/team1/foo`

1. Gateway receives request on port 8080
2. Host is "parent2.com" -> matches parent2 route
3. parent2 delegates to svc1 and svc2
4. Path starts with "/anything/team1" -> matches svc1's rule
6. Traffic goes to httpbin service

Policy merge:
  - svc1 has: origin=svc1    (child policy)
  - parent2 has: origin=parent2  (parent policy)
  - parent2 annotation: DeepMergePreferParent
  - Result: origin=parent2

These are the expected results:

| url | merge  | who won | expected origin header |
|---|---|---|---|
| `parent1.com/anything/team1` | `DeepMergePreferChild` | child wins | `svc1` |
| `parent1.com/anything/team2` | `DeepMergePreferChild` | child wins | `svc2` |
| `parent2.com/anything/team1` | `DeepMergePreferParent` | parent wins | `parent2` |
| `parent2.com/anything/team2` | `DeepMergePreferParent` | parent wins | `parent2` |

---

ACtual Behavior:

Results are non-deterministic. sometimes `parent2.com/anything/team1` returns `origin: svc1` instead of `origin: parent2`, or vice versa.

# Bug (I think I got it)

```go
func (a *AttachedPolicies) AppendWithPriority(HierarchicalPriority int, l ...AttachedPolicies) {
    for _, l := range l {
        for k, v := range l.Policies {
            for j := range v {
                v[j].HierarchicalPriority = HierarchicalPriority  // <- this line
            }
            a.Policies[k] = append(a.Policies[k], v...)
        }
    }
}
```

we are updating the `v[j].HierarchicalPriority` in place , so it changes the data for any other delegation chain that reads from the same slice.

`for k, v := range l.Policies {` - v here is a copy of the slice header (pointer + len + cap), but the underlying array is shared.

```
for j := range v {
    v[j].HierarchicalPriority = HierarchicalPriority  // ← mutates shared array
}
```

v[j] indexes into the underlying array directly - updating this stuff affect the original data.

