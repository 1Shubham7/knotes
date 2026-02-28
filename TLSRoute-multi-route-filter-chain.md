### Port

The Gateway is listening on port :8443. That's the one "apartment" where all TLS traffic arrives. Multiple routes (TLSRoute, TCPRoute) all share this same door.


### Listener

The Listener is a doorman at port 8443 with specific instructions about who to let in. A Listener is a configuration on your Gateway that says:

- "I am waiting for connections on this port"
- "I expect this type of traffic"
- "I only serve these hostnames". hostname is the domain name the client is trying to reach

```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: Gateway
metadata:
  name: my-gateway
spec:
  listeners:
    - name: tls-listener
      port: 8443
      protocol: TLS
      hostname: "*.example.com"   # only serve example.com subdomains
      tls:
        mode: Passthrough          # don't decrypt, just forward
```

So if a client connects to port 8443 saying:
```
"I want foo.example.com"     ✅ matches *.example.com → let in
"I want foo.other.com"       ❌ doesn't match         → rejected
"I want bar.example.com"     ✅ matches *.example.com → let in
```


Two ways a gateway can handle TLS traffic:

**Terminate mode** — Gateway decrypts the traffic, reads it, re-encrypts, forwards.
```
Client → [encrypted] → Gateway DECRYPTS → reads request → re-encrypts → Backend
```

**Passthrough mode** — Gateway never decrypts. Forwards raw encrypted bytes.
```
Client → [encrypted] → Gateway reads only SNI → forwards [still encrypted] → Backend
```

Passthrough is used when:

- The backend needs to handle its own certificates
- You don't want the gateway to ever see the unencrypted data (security/compliance)
- You're routing based purely on hostname, not on HTTP paths or headers

The Problem This Creates:

In Passthrough mode, the gateway is blind to everything inside the encrypted packet. It cannot see:

- HTTP headers
- URL paths
- Cookies
- Anything inside the TLS envelope

The only thing it can see is the SNI — the hostname the client announces before encryption kicks in.

### SNI (Server Name Indication)

Imagine you're sending a sealed letter. The envelope is visible to everyone — it has the recipient's name and address written on the outside. The letter inside is private — nobody can read it except the recipient.
SNI is the envelope. Everything else in TLS is the sealed letter.

Before SNI existed, there was a problem:
One server, one IP address, one port — but maybe multiple websites hosted on it. When a client connects, the server needs to know which website's certificate to present. But the client hasn't told it yet who it wants.
SNI solves this: the client announces the hostname first, before encryption starts, so the server (or gateway) can react appropriately.

### Filter Chain

Remember our hotel doorman (Listener) at port 8443? He lets guests in based on *.example.com. Now the guests are inside the building.
Inside there's a row of greeters. Each greeter has a checklist:
Greeter 1: "Are you here for foo.example.com? Yes? Come with me → Backend A"
Greeter 2: "Are you here for bar.example.com? Yes? Come with me → Backend B"  
Greeter 3: "Anyone? No checklist. Follow me → Backend C"
Each greeter = a Filter Chain. Envoy goes through them in order, top to bottom, and picks the first one that matches.

A Filter Chain is a rule inside an Envoy listener that says:
- Match condition (e.g. SNI = foo.example.com)
- where to forward the matching traffic (which backend)

E.g.
```
filter_chains:
  - filter_chain_match:
      server_names: ["foo.example.com"]   # match condition
    filters:
      - name: tcp_proxy
        typed_config:
          cluster: backend-a              # where to forward
```

One Envoy listener (port) can have multiple filter chains. When a connection arrives, Envoy checks each filter chain's match condition and picks the most specific one that matches.

filter_chains:
  - filter_chain_match:
      server_names: ["foo.example.com"]   # match condition
    filters:
      - name: tcp_proxy
        typed_config:
          cluster: backend-a              # where to forward

  - filter_chain_match:
      server_names: ["bar.example.com"]   # match condition
    filters:
      - name: tcp_proxy
        typed_config:
          cluster: backend-b              # where to forward

  - filter_chain_match: {}                # no condition = catch-all
    filters:
      - name: tcp_proxy
        typed_config:
          cluster: backend-c
```

---

### The "most specific wins" rule

This is critical. Envoy doesn't just pick the first match — it picks the **most specific** match.
```
Incoming SNI: "foo.example.com"

Filter Chain A: server_names: ["foo.example.com"]   ← exact match
Filter Chain B: server_names: ["*.example.com"]     ← wildcard match
Filter Chain C: {}                                   ← catch-all

Envoy picks: Filter Chain A (most specific)

Even if catch-all is checked first, exact match wins. This is why **ordering matters** when you write the config — but also why specificity matters more than order.

---

### Now — the bug in your current code

Your current code does this:

```
Route A → foo.example.com  →  Filter Chain 1: server_names: ["foo.example.com"]
Route B → foo.example.com  →  Filter Chain 2: server_names: ["foo.example.com"]
```

Two filter chains, same SNI. What does Envoy do?
```
Incoming SNI: "foo.example.com"

Filter Chain 1: server_names: ["foo.example.com"]  ← Envoy picks this, ALWAYS
Filter Chain 2: server_names: ["foo.example.com"]  ← NEVER reached, ever

BUG 2

one hostname per route. Your current code also does this for a TLSRoute with multiple hostnames:
```
# TLSRoute declares:
hostnames:
  - foo.example.com
  - bar.example.com
```

Your code picks only the first/most-specific hostname and creates:
```yaml
yamlfilter_chain_match:
  server_names: ["foo.example.com"]   # bar.example.com is just dropped!
```

So bar.example.com traffic never gets routed. Silently broken again. The fix is simple — put all hostnames in one filter chain:

### TCP vs TLS routes

