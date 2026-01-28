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






















