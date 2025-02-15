# Capsule Proxy

This project is an add-on for [Capsule](https://github.com/clastix/capsule), the operator providing multi-tenancy in Kubernetes.

## The problem

Kubernetes RBAC cannot list only the owned cluster-scoped resources since there are no ACL-filtered APIs. For example:

```
$ kubectl get namespaces
Error from server (Forbidden): namespaces is forbidden:
User "alice" cannot list resource "namespaces" in API group "" at the cluster scope
```

However, the user can have permissions on some namespaces

```
$ kubectl auth can-i [get|list|watch|delete] ns oil-production
yes
```

The reason, as the error message reported, is that the RBAC _list_ action is available only at Cluster-Scope and it is not granted to users without appropriate permissions.

To overcome this problem, many Kubernetes distributions introduced mirrored custom resources supported by a custom set of ACL-filtered APIs. However, this leads to radically change the user's experience of Kubernetes by introducing hard customizations that make it painful to move from one distribution to another.

With **Capsule**, we took a different approach. As one of the key goals, we want to keep the same user's experience on all the distributions of Kubernetes. We want people to use the standard tools they already know and love and it should just work.

## How it works

This project is an add-on of the main [Capsule](https://github.com/clastix/capsule) operator, so make sure you have a working instance of Caspule before attempting to install it. Use the `capsule-proxy` only if you want Tenant Owners to list their own Cluster-Scope resources.

The `capsule-proxy` implements a simple reverse proxy that intercepts only specific requests to the APIs server and Capsule does all the magic behind the scenes.

Current implementation only filter two type of requests:

* `api/v1/namespaces`
* `api/v1/nodes`
* `apis/storage.k8s.io/v1/storageclasses`
* `apis/networking.k8s.io/{v1,v1beta1}/ingressclasses`

All other requests are proxied transparently to the APIs server, so no side effects are expected. We're planning to add new APIs in the future, so PRs are welcome!

## Installation

The `capsule-proxy` can be deployed in standalone mode, e.g. running as a pod bridging any Kubernetes client to the APIs server. Optionally, it can be deployed as a sidecar container in the backend of a dashboard.

An Helm Chart is available [here](./charts/capsule-proxy/README.md).

## Does it work with kubectl?

Yes, it works by intercepting all the requests from the `kubectl` client directed to the APIs server. It works with both users who use the TLS certificate authentication and those who use OIDC.

As tenant owner `alice`, you can use `kubectl` to create some namespaces:
```
$ kubectl --context alice-oidc@mycluster create namespace oil-production
$ kubectl --context alice-oidc@mycluster create namespace oil-development
$ kubectl --context alice-oidc@mycluster create namespace gas-marketing
```

and list only those namespaces:

```
$ kubectl --context alice-oidc@mycluster get namespaces
NAME                STATUS   AGE
gas-marketing       Active   2m
oil-development     Active   2m
oil-production      Active   2m
```

## Does it work with my preferred Kubernetes dashboard?

If you're using a client-only dashboard, for example [Lens](https://k8slens.dev/), the `capsule-proxy` can be used as with `kubectl` since this dashboard usually talks to the APIs server using just a `kubeconfig` file.

![Lens dashboard](assets/images/lens.png)

For a web-based dashboard, like the [Kubernetes Dashboard](https://github.com/kubernetes/dashboard), the `capsule-proxy` can be deployed as a sidecar container in the backend, following the well-known cloud-native _Ambassador Pattern_.

![Kubernetes dashboard](assets/images/kubernetes-dashboard.png)

## Documentation

You can find more detailed documentation [here](https://github.com/clastix/capsule/blob/master/docs/index.md).

## Contributions

This is an open-source software released with Apache2 [license](./LICENSE). Feel free to open issues and pull requests. You're welcome!

## How to

### Run locally for test and debug

This guide helps new contributors to locally debug in _out or cluster_ mode the project.

1. You need to run a kind cluster and find the endpoint port of `kind-control-plane` using `docker ps`:

```bash
❯ docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED          STATUS          PORTS                       NAMES
88432e392adb   kindest/node:v1.20.2   "/usr/local/bin/entr…"   32 seconds ago   Up 28 seconds   127.0.0.1:64582->6443/tcp   kind-control-plane
```

2. You need to generate TLS cert keys for localhost, you can use [mkcert](https://github.com/FiloSottile/mkcert):

```bash
> cd /tmp
> mkcert localhost
> ls
localhost-key.pem localhost.pem
```

3. Run the proxy with the following options

```bash
go run main.go --ssl-cert-path=/tmp/localhost.pem --ssl-key-path=/tmp/localhost-key.pem --k8s-control-plane-url=https://localhost:<KIND PORT> --enable-ssl=true --kubeconfig=<YOUR KUBERNETES CONFIGURATION FILE>
```

5. Edit the KUBECONFIG file (you should make a copy and work on it) as follows:
- Find the section of your cluster
- replace the server path with `https://127.0.0.1:9001`
- replace the certificate-authority-data path with the content of your rootCA.pem file. (if you use mkcert, you'll find with `cat "$(mkcert -CAROOT)/rootCA.pem"|base64|tr -d '\n'`)

6. Now you should be able to run kubectl using the proxy!

### Debug in a remote Kubernetes cluster

In some cases, you would need to debug the in-cluster mode and [`delve`](https://github.com/go-delve/delve) plays a big role here.

1. build the Docker image with `delve` issuing `make dlv-build`
2. with the `quay.io/clastix/capsule-proxy:dlv` produced Docker image, publish it or load it to your [KinD](https://github.com/kubernetes-sigs/kind) instance (`kind load docker-image --name capsule --nodes capsule-control-plane quay.io/clastix/capsule-proxy:dlv`)
3. change the Deployment image using `kubectl edit` or `kubectl set image deployment/capsule-proxy capsule-proxy=quay.io/clastix/capsule-proxy:dlv`
4. wait for the image rollout (`kubectl -n capsule-system rollout status deployment/capsule-proxy`)
5. perform the port-forwarding with `kubectl -n capsule-system port-forward $(kubectl -n capsule-system get pods -l app.kubernetes.io/name=capsule-proxy --output name) 2345:2345`
6. connect using your `delve` options

> _Nota Bene_: the application could be killed by the Liveness Probe since delve will wait for the debugger connection before starting it.
> Feel free to edit the remove the probes to avoid this kind of issue.
