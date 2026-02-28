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
