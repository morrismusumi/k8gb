# k8gb Local Demo

A local, fully reproducible demo of [k8gb](https://www.k8gb.io/) (Kubernetes Global Balancer) running across two k3d clusters (`east` and `west`). Everything beyond the base clusters is installed through GitOps with Flux.

## What this demo sets up

- **Two k3d clusters**, `east` and `west`, each acting as a separate "region".
- **kube-vip** in each cluster to provide `LoadBalancer` IPs (the default k3s `servicelb` is disabled).
- **k8gb** in each cluster, which does DNS-based global load balancing and failover between the two clusters.
- **A sample web app** deployed to both clusters, exposed through Traefik.
- **Unbound** (`edge-dns`) acting as the upstream/edge DNS server that clients query.
- **Netshoot** (`netshoot`) as a test client for running `dig`, `curl`, etc. from inside the Docker networks.

## Architecture

| Component | Network | Address range |
|---|---|---|
| `east` cluster | `k3d-east` | `172.19.0.0/16` |
| `west` cluster | `k3d-west` | `172.20.0.0/16` |
| `edge-dns` and `netshoot` | `k8gb_dns-network` | `172.28.0.0/24` |

The Traefik ingress in each cluster gets a fixed LoadBalancer IP from kube-vip:

| Cluster | Traefik IP |
|---|---|
| `east` | `172.19.0.50` |
| `west` | `172.20.0.50` |

How it fits together: a client (netshoot) asks `edge-dns` (Unbound) for the demo hostname. Unbound forwards the query to the k8gb-managed CoreDNS in the clusters, and k8gb answers with the IP(s) of whichever cluster(s) currently have healthy app endpoints.

## Prerequisites

- Docker and Docker Compose
- [k3d](https://k3d.io/)
- `kubectl`
- [Flux CLI](https://fluxcd.io/flux/installation/)
- An SSH key with write access to your fork/copy of this repo (the examples use `$HOME/.ssh/id_rsa`)

## Step by step

### 1. Create the clusters

k3s ships with a built-in service load balancer (`servicelb`). It is disabled here so that kube-vip can hand out LoadBalancer IPs instead.

```bash
k3d cluster create east --k3s-arg "--disable=servicelb@server:*"
k3d cluster create west --k3s-arg "--disable=servicelb@server:*"
```

Each cluster gets its own Docker network (`k3d-east`, `k3d-west`) and a kubeconfig context (`k3d-east`, `k3d-west`).

### 2. Start the DNS server and test client

The `compose.yml` defines two containers:

- `edge-dns`: an Unbound DNS server that sits in front of the clusters.
- `netshoot`: a toolbox container to test from.

```bash
docker compose up -d
```

This also creates the `k8gb_dns-network` network (`172.28.0.0/24`).

### 3. Connect the networks

By default each Docker network is isolated. k8gb needs the clusters to reach each other, and the DNS server and client need to reach both clusters. These commands attach containers to additional networks so every node can talk to every other.

**East and west clusters to each other:**

```bash
docker network connect k3d-east k3d-west-server-0
docker network connect k3d-east k3d-west-serverlb
docker network connect k3d-west k3d-east-server-0
docker network connect k3d-west k3d-east-serverlb
```

**Cluster nodes to the DNS network** (so the clusters can reach `edge-dns`):

```bash
docker network connect k8gb_dns-network k3d-east-server-0
docker network connect k8gb_dns-network k3d-west-server-0
```

**Edge DNS and test client to both clusters:**

```bash
docker network connect k3d-east edge-dns
docker network connect k3d-west edge-dns
docker network connect k3d-east netshoot
docker network connect k3d-west netshoot
```

### 4. Bootstrap Flux on both clusters

Flux is the only thing installed by hand. It then installs everything else (kube-vip, k8gb and the web app) from the Git repo. Each cluster watches its own path under `fleet-infra/clusters/`.

**East:**

```bash
flux bootstrap git \
  --url=ssh://git@github.com/morrismusumi/k8gb.git \
  --branch=main \
  --path=./fleet-infra/clusters/east \
  --context=k3d-east \
  --private-key-file=$HOME/.ssh/id_rsa
```

**West:**

```bash
flux bootstrap git \
  --url=ssh://git@github.com/morrismusumi/k8gb.git \
  --branch=main \
  --path=./fleet-infra/clusters/west \
  --context=k3d-west \
  --private-key-file=$HOME/.ssh/id_rsa
```

Once reconciled, each cluster has **kube-vip**, **k8gb** and the **web-app** installed. You can watch progress with:

```bash
flux get kustomizations --context k3d-east
flux get kustomizations --context k3d-west
```

### 5. Pin the Traefik LoadBalancer IPs

Traefik is the ingress controller bundled with k3s. Annotate its service so kube-vip assigns it a known, predictable IP in each cluster:

```bash
# east
kubectl --context k3d-east patch svc traefik -n kube-system \
  -p '{"metadata":{"annotations":{"kube-vip.io/loadbalancerIPs":"172.19.0.50"}}}'

# west
kubectl --context k3d-west patch svc traefik -n kube-system \
  -p '{"metadata":{"annotations":{"kube-vip.io/loadbalancerIPs":"172.20.0.50"}}}'
```

These fixed IPs are what k8gb returns in DNS answers, and what Unbound is expected to resolve to.

### 6. Verify (suggested)

Check that the Traefik services picked up their IPs:

```bash
kubectl --context k3d-east get svc traefik -n kube-system
kubectl --context k3d-west get svc traefik -n kube-system
```

Then query from the test client. Replace `<your-hostname>` with the host defined in the web-app's Gslb resource:

```bash
docker exec -it netshoot dig <your-hostname>
```

To see failover, scale the web app down to zero in one cluster and query again. The answer should only contain the other cluster's IP:

```bash
kubectl --context k3d-east scale deploy <web-app-deployment> --replicas=0
docker exec -it netshoot dig <your-hostname>
```

## Cleanup

```bash
docker compose down
k3d cluster delete east west
```

## Troubleshooting

- **Traefik service stuck in `<pending>`**: check that kube-vip is running in the cluster and that the annotation in step 5 was applied.
- **Clusters can't reach each other**: re-check the `docker network connect` commands in step 3. Already-connected errors are harmless.
- **Flux bootstrap fails with an SSH error**: confirm the deploy key/SSH key has write access and that the repo URL uses the `ssh://git@github.com/<owner>/<repo>.git` form.
