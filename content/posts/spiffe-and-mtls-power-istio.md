---
title: How SPIFFE and mTLS Power Zero Trust in Istio
---

TLS is integral to sending data securely over a network. Istio talks a lot about mTLS, workload identity and the mythical _zero trust network_. I vaguely knew what these meant, but I needed to get a deeper understanding to really get the full picture—so I decided to take a dip into the deep end and see what all the fuss was about. Before diving into Istio specifics though, I needed to understand what TLS was and how it worked.
### What is TLS
Funny enough, the name _Transport Layer Security_ is a bit of a misnomer. It actually doesn't operate at the transport layer (L4) at all; instead, it sits above it and protects communication at the application layer (L7). TLS works through a well established "handshake" between the two parties.

Before a certificate exchange occurs, there's a "ClientHello/ServerHello" initiation where the two sides exchange things like TLS versions, cipher suites, and also the client might include the Subject Name Indicator (SNI) to tell the server which domain it wants to connect to so that it can receive the correct certificate in response.

> [!aside]
>SNI is especially important in scenarios where you're dealing with a proxy listening on an IP that routes traffic to multiple backends—like an ingress. A client says it wants to talk to `foo.app`, so the proxy will use that SNI value to forward the request to the IP of `foo.app`.  

Next, the server sends its certificate to the client. This certificate includes:
- The authority (CA) that signed it, along with a verification signature created using the CA's private key. The client can check its authenticity by using the CA's public key.
- The expiration date and time.
- The server's public key—to be used for the handshake until symmetric key is established.
- The Subject Alternate Name (SAN)—which lists the domain(s) that this certificate is valid for.

Once the client verifies the authenticity of the certificate, the client and server exchange ephemeral key shares, from which they derive a shared symmetric key. All future traffic is encrypted using this key because it's faster.

The key exchange is interesting. Both sides generate a public and private key. They give the public one out to the world to encrypt data sent to them and they decrypt it using their private key. Using these keys is expensive—the algorithms used are slow and CPU intensive. Symmetric key encryption is orders of magnitude faster, so the two sides generate a _shared symmetric key_ by using the other's public key and their own private key. The algorithm(s) behind the key generation is kind of gnarly so I won't get into that. 
### Let's Talk Istio
TLS is very important for service meshes, more precisely, _mutual TLS_ (or mTLS). The difference is that now both sides (client and server) exchange certificates, emphasizing the "zero trust network" thing.

Workloads in service meshes are typically no more than deployments living in namespaces, so the question becomes: How do these workloads get certificates generated that prove their identity?

In enters **Secure Protection Identity Framework For Everyone**—that's a mouthful so they call it SPIFFE instead. Just think of it as a unique identifier for a workload or a set of common workloads. The actual value is made up of the service account name and namespace with a few extra bits added.

Here we see the SPIFFE ID for the service account `foo` in the `bar` namespace:

```
spiffe://cluster.local/ns/bar/sa/foo
```

> [!important] 
> At this point you might be thinking "Wait a second—How is this unique?" Well, it can be if you want it to be. The namespace is the default trust boundary, but you can easily create a service account per workload and achieve finer grained identities.

 When istiod signs a certificate for a workload, it will add the SPIFFE ID in the SAN field of the certificate. What this means is that the certificate is  unique to that workload (or workloads depending on your service account scope), this is why they call it a _workload identity_.

Now I want to switch gears a bit and talk about how Istio disperses these certificates with unique identities. 

### Istiod and istio-agent
[istio-agent](https://github.com/istio/istio/blob/master/architecture%2Fsecurity%2Fistio-agent.md) is a local Secret Discovery Service (SDS) process that runs along side the main`istio-proxy`. The entire job of `istio-agent` is secret management and communicating with `istiod` to request and return certificates to `istio-proxy`. `istio-agent` is the only component out of the bunch to get its own isolated process, specifically to keep secret management out of `istio-proxy`.

Every pod that's created automatically comes with a service account token baked in. It's also signed by the Kubernetes API—making it a great way to verify the authenticity of a workload. `istio-agent` will take that service account token, slap it on a certificate signing request (CSR) and send it to `istiod`. `istiod` sees this token and extrapolates the important information to create that SPIFFE ID then it sends the certificate back to `istio-agent`.

```bash
$ cat var/run/secrets/kubernetes.io/serviceaccount/token
{
  "iss": "https://kubernetes.default.svc.cluster.local",
  "kubernetes.io/serviceaccount/namespace": "bar",
  "kubernetes.io/serviceaccount/service-account.name": "foo",
  "kubernetes.io/serviceaccount/service-account.uid": "1234abcd-5678-efgh-ijkl-1234567890mn",
  "sub": "system:serviceaccount:bar:foo",
  "exp": 1719439200,
  ...
}
```

This service account approach also solves a major "chicken-and-egg" problem that occurs with certificate issuance. Normally, you need a certificate to authenticate before getting one. In Istio—a newly created workload doesn't have any certificates unless you specifically baked them in—so instead, they use the Kubernetes service account token as a way to prove their identity.

So there you have it. That’s how Istio leverages TLS, SPIFFE, and Kubernetes primitives to achieve a zero-trust environment. Hopefully that clears up some of the confusing terminology you often encounter when you're deep in the weeds of a service mesh. 






