+++
title = "Automatic mTLS"
description = "Linkerd automatically enables mutual Transport Layer Security (TLS) for all communication between meshed applications."
weight = 4
aliases = [
  "/2/features/automatic-tls"
]
+++

By default, Linkerd automatically enables mutual Transport Layer Security
(mTLS) for most TCP traffic between meshed pods, by establishing and
authenticating secure, private TLS connections between Linkerd proxies.
This means that Linkerd can add authenticated, encrypted communication to your
application with very little work on your part. And because the Linkerd control
plane also runs on the data plane, this means that communication between
Linkerd's control plane components are also automatically secured via mTLS.

Not all traffic can be automatically mTLS'd, but it's easy to [verify which
traffic is](/2/tasks/securing-your-service/). See [Caveats and future
work](#caveats-and-future-work) below for details on which traffic cannot
currently be automatically encrypted.

## How does it work?

In short, Linkerd's control plane issues TLS certificates to proxies, which are
scoped to the [Kubernetes
ServiceAccount](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
of the containing pod and automatically rotated every 24 hours. The proxies use
these certificates to encrypt and authenticate TCP traffic to other
proxies.

To do this, Linkerd maintains a set of credentials in the cluster: a trust
anchor, and an issuer certificate and private key. These credentials are
generated by Linkerd itself during install time, or optionally provided by an
external source, e.g. [Vault](https://vaultproject.io) or
[cert-manager](https://github.com/jetstack/cert-manager). The issuer
certificate and private key are placed into a [Kubernetes
Secret](https://kubernetes.io/docs/concepts/configuration/secret/). By default,
the Secret is placed in the `linkerd` namespace and can only be read by the
service account used by the [Linkerd control
plane](/2/reference/architecture/)'s `identity` component.

On the data plane side, each proxy is passed the trust anchor in an environment
variable. At startup, the proxy generates a private key, stored in a [tmpfs
emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir) which
stays in memory and never leaves the pod. The proxy connects to the control
plane's `identity` component, validating the connection to `identity` with the
trust anchor, and issues a [certificate signing request
(CSR)](https://en.wikipedia.org/wiki/Certificate_signing_request). The CSR
contains an initial certificate with identity set to the pod's [Kubernetes
ServiceAccount](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/),
and the actual service account token, so that `identity` can validate that the
CSR is valid. After validation, the signed trust bundle is returned to the
proxy, which can use it as both a client and server certificate. These
certificates are scoped to 24 hours and dynamically refreshed using the
same mechanism.

When a proxy injected into a pod receives an outbound connection from the
application container, it performs service discovery by looking up that IP
address with the Linkerd control plane. When the destination is in the
Kubernetes cluster, the control plane provides the proxy with the destination's
endpoint addresses, along with metadata. When an identity name is included in
this metadata, this indicates to the proxy that it can initiate mutual TLS. When
the proxy connects to the destination, it initiates a TLS handshake, verifying
that the destination proxy's certificate was signed for the expected identity
name.

## Maintenance

The trust anchor generated by `linkerd install` expires after 365 days, and
must be [manually
rotated](/2/tasks/manually-rotating-control-plane-tls-credentials/).
Alternatively you can [provide the trust anchor
yourself](/2/tasks/generate-certificates/) and control the expiration date.

By default, the issuer certificate and key are not automatically rotated. You
can [set up automatic rotation with
`cert-manager`](/2/tasks/automatically-rotating-control-plane-tls-credentials/).

## Caveats and future work

There are a few known gaps in Linkerd's ability to automatically encrypt and
authenticate all communication in the cluster. These gaps will be fixed in
future releases:

* Linkerd does not currently *enforce* mTLS. Any unencrypted requests inside
  the mesh will be opportunistically upgraded to mTLS. Any requests originating
  from inside or outside the mesh will not be automatically mTLS'd by Linkerd.
  This will be addressed in a future Linkerd release, likely as an opt-in
  behavior as it may break some existing applications.

* Ideally, the ServiceAccount token that Linkerd uses would not be shared with
  other potential uses of that token. In future Kubernetes releases, Kubernetes
  will support audience/time-bound ServiceAccount tokens, and Linkerd will use
  those instead.
