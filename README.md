This repository contains the configuration for my personal Kubernetes cluster.

## Getting this running
To bootstrap anew:
1. provision a cluster (I used [k3s](https://k3s.io/) with `--disable=traefik`)
2. install [Argo CD](https://argo-cd.readthedocs.io/en/stable/getting_started/)
3. sign in and click "new app" then "edit as YAML"
4. paste in the contents of [apps.yaml](apps/apps.yaml)

## Cluster overview

I have two Kubernetes nodes at the moment.
One of them is on my [**Bitfolk**](https://bitfolk.com) VPS, which is your standard
Debian server running in a datacentre somewhere. Its hostname is `saraneth`.

The other is a [**Raspberry Pi 5** 8GB](https://www.raspberrypi.com/products/raspberry-pi-5/)
running on my desk. Its hostname is `pi`.

They&rsquo;re connected together via a [**Tailscale**](https://tailscale.com) mesh
virtual private network using the k3s [experimental integration](https://docs.k3s.io/networking/distributed-multicloud#integration-with-the-tailscale-vpn-provider-experimental).

As my VPS already had a PostgreSQL server running, we&rsquo;re using that for
the cluster datastore. The `sareneth` node has a
`CriticalAddonsOnly=true:PreferNoSchedule` taint to try and push as much work
onto the Raspberry Pi as possible; the VPS&rsquo;s job is to run the control plane
and serve the ingress traffic to the outside world.

## Projects

### Infrastructure

We&rsquo;re using [**k3s**](https://k3s.io/) for our Kubernetes distribution.
I picked it as it was lightweight enough to run on my VPS alongside all the
other non-Kubernetes things my VPS is doing, while being portable enough that
I could feasibly run it on any hardware I might want to extend this cluster to
in the future. It also came bundled with some integrations out the box that made
my life easier.

We&rsquo;re running [**Argo CD**](https://argo-cd.readthedocs.io/en/stable/),
which is used to facilitate &ldquo;declarative GitOps&rdquo;: when you push
some YAML code to this repository, Argo CD will notice the change and will take
action to make sure the state of the Kubernetes cluster aligns with the code
you wrote.

We are using [**Ingress-Nginx**](https://kubernetes.github.io/ingress-nginx/) and
[**cert-manager**](https://cert-manager.io/) to expose our HTTP/HTTPS services to
the outside world. As we&rsquo;re using this instead of Traefik, the ingress
controller that is bundled with k3s, we can install k3s with the
`--disable=traefik` flag to save some overhead. I chose to use Ingress-Nginx as
I was already familiar with it.

For volumes and storage, we&rsquo;re using [**Longhorn**](https://longhorn.io/),
which means that any persistent storage used by our containers will, by default,
be replicated in both nodes. As I want to run some applications that have heavy
read/write activity, I have configured a StorageClass that removes SD cards from
the pool where needed.

## End-user services

We&rsquo;re running a single-user [**Mastodon**](https://joinmastodon.org/) instance on
Kubernetes. This used to run solely on my VPS, but this caused problems: it is
a disconcertingly resource-intensive Ruby program, and it meant my VPS often
ran out of RAM and fell offline halfway through an upgrade.

I&rsquo;m using a hybrid solution at the moment. The pods are running in Kubernetes,
but I&rsquo;m still using the old PostgreSQL and Redis servers that were already on
my VPS for the "important" stuff. Instead of using Kubernetes persistent volumes,
we&rsquo;re using [Google Cloud Storage](https://cloud.google.com/storage) for
user data.

## Fault tolerance

I can reasonably expect my VPS to be up and running at all times.
It&rsquo;s in a datacentre backed by proper hardware.
It is comparatively resource-bound, though: we&rsquo;re low on disk space and have
contention for CPU with other Bitfolk tenants.

My Raspberry Pi cannot be trusted to be as reliable.
I want to be able to unplug the Raspberry Pi to move things around in my office
with no notice, or the MicroSD card upon which it runs to fail at any moment.
However, the storage is cheap and all the hardware is available for my sole use.

As such:
* having the cluster data store only on the VPS is fine
* any persistent storage on the Pi is must also be stored on the VPS
* the node on the VPS has a `CriticalAddonsOnly=true:PreferNoSchedule` taint to
  encourage pods to be scheduled elsewhere if possible, while still allowing it
  to step in if needed (which `…:NoSchedule` would prevent)


## Costs

The Raspberry Pi cost me £90 (2023), and its 256GB MicroSD card cost £30 (2025).
Google Cloud storage, used for Mastodon,  costs me about £1 a month.
A base plan on my VPS costs £65 a year, and I pay extra for some additional RAM.

I&rsquo;m using the personal Tailscale plan, which is free.
