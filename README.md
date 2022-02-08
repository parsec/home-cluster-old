<div align="center">

<img src="https://camo.githubusercontent.com/5b298bf6b0596795602bd771c5bddbb963e83e0f/68747470733a2f2f692e696d6775722e636f6d2f7031527a586a512e706e67" align="center" width="144px" height="144px"/>

### My home operations repository :octocat:

_... managed with Flux, Renovate and GitHub Actions_ :robot:

</div>

<br/>

<div align="center">

[![Discord](https://img.shields.io/discord/673534664354430999?style=for-the-badge&label=discord&logo=discord&logoColor=white)](https://discord.gg/k8s-at-home)
[![k3s](https://img.shields.io/badge/k3s-v1.22.6-brightgreen?style=for-the-badge&logo=kubernetes&logoColor=white)](https://k3s.io/)
[![pre-commit](https://img.shields.io/badge/pre--commit-enabled-brightgreen?logo=pre-commit&logoColor=white&style=for-the-badge)](https://github.com/pre-commit/pre-commit)
[![GitHub Workflow Status](https://img.shields.io/github/workflow/status/parsec/home-cluster/Schedule%20-%20Renovate%20Helm%20Releases?label=renovate&logo=renovatebot&style=for-the-badge)](https://github.com/parsec/home-cluster/actions/workflows/renovate-schedule.yaml)
[![Lines of code](https://img.shields.io/tokei/lines/github/parsec/home-cluster?style=for-the-badge&color=brightgreen&label=lines&logo=codefactor&logoColor=white)](https://github.com/parsec/home-cluster/graphs/contributors)

</div>

---

## 📖 Overview

This is my home cluster, built on [k3s](https://k3s.io), backed by [FluxCD](https://fluxcd.io/), [Terraform](https://terraform.io), [Ansible](https://www.ansible.com/), and GitOps/DevOps principles. This is a place for me to learn, and share everything I've learned along the way. Feel free to ask questions, submit improvements, and learn from all the hard work I've put into this!

If you want to read about my adventures, I try to post about all my learning experiences on [my blog](https://blog.gitgud.sh). Come git gud with me!

---

## ⛵ Kubernetes

There's an excellent template over at [k8s-at-home/template-cluster-k3](https://github.com/k8s-at-home/template-cluster-k3s) that I based my cluster on if you feel like following along :)

### Installation

My cluster is [k3s](https://k3s.io/) provisioned overtop bare-metal Ubuntu 20.04 using the [Ansible](https://www.ansible.com/) galaxy role [ansible-role-k3s](https://github.com/PyratLabs/ansible-role-k3s). This is a semi hyper-converged cluster, workloads and block storage are sharing the same available resources on my nodes while I have a separate server for (NFS) file storage.

🔸 _[Click here](./ansible/) to see my Ansible playbooks and roles._

### Core Components

- [mozilla/sops](https://toolkit.fluxcd.io/guides/mozilla-sops/): Manages secrets for Kubernetes, Ansible and Terraform.
- [kubernetes-sigs/external-dns](https://github.com/kubernetes-sigs/external-dns): Automatically manages DNS records from my cluster in a cloud DNS provider.
- [jetstack/cert-manager](https://cert-manager.io/docs/): Creates SSL certificates for services in my Kubernetes cluster.
- [traefik/traefik](https://github.com/traefik/traefik): Ingress controller to expose HTTP traffic to pods over DNS.
- [kubernetes-sigs/nfs-subdir-external-provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner): Used for provisioning NFS storage on my NAS.

### GitOps

[Flux](https://github.com/fluxcd/flux2) watches my [cluster](./cluster/) folder (see Directories below) and makes the changes to my cluster based on the YAML manifests.

[Renovate](https://github.com/renovatebot/renovate) watches my **entire** repository looking for dependency updates, when they are found a PR is automatically created. When some PRs are merged [Flux](https://github.com/fluxcd/flux2) applies the changes to my cluster.

### Directories

The Git repository contains the following directories under [cluster](./cluster/) and are ordered below by how [Flux](https://github.com/fluxcd/flux2) will apply them.

- **base**: directory is the entrypoint to [Flux](https://github.com/fluxcd/flux2).
- **crds**: directory contains custom resource definitions (CRDs) that need to exist globally in your cluster before anything else exists.
- **core**: directory (depends on **crds**) are important infrastructure applications (grouped by namespace) that should never be pruned by [Flux](https://github.com/fluxcd/flux2).
- **apps**: directory (depends on **core**) is where your common applications (grouped by namespace) could be placed, [Flux](https://github.com/fluxcd/flux2) will prune resources here if they are not tracked by Git anymore.

### Networking

| Name                                         | CIDR              |
|----------------------------------------------|-------------------|
| Kubernetes Nodes                             | `10.1.0.0/24`     |
| Kubernetes external services (Calico w/ BGP) | `10.117.0.0/24`   |
| Kubernetes pods                              | `10.69.0.0/16`    |
| Kubernetes services                          | `10.96.0.0/16`    |

### Persistent Volume Data Backup and Recovery

Currently, I use [nfs-subdir-external-provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner) to provision PVs on my NAS over NFS. This way, at least for the time being, my PVs are backed up along with the NAS in Google Workspace (Google Drive). This isn't the most resilient or appropriate way to do it, but I'm not quite ready to set up block storage yet :)

---

## 🌐 DNS

### Ingress Controller

Over WAN, I have port forwarded ports `80` and `443` to the load balancer IP of my ingress controller that's running in my Kubernetes cluster.

[Cloudflare](https://www.cloudflare.com/) works as a proxy to hide my homes WAN IP and also as a firewall. When not on my home network, all the traffic coming into my ingress controller on port `80` and `443` comes from Cloudflare.

🔸 _Cloudflare is also configured to GeoIP block all countries except a few I have whitelisted_

### Internal DNS

For now, this is just handled by my Mikrotik RB4011 router (soon to upgrade to an RB5009!)

I will, at some point in the future, do something a little more exciting with this!

### External DNS

[external-dns](https://github.com/kubernetes-sigs/external-dns) is deployed in my cluster and configure to sync DNS records to [Cloudflare](https://www.cloudflare.com/). The only ingresses `external-dns` looks at to gather DNS records to put in `Cloudflare` are ones that I explicitly set an annotation of `external-dns/is-public: "true"`

🔸 _[Click here](./terraform/cloudflare) to see how else I manage Cloudflare._

### Dynamic DNS

My home IP can change at any given time and in order to keep my WAN IP address up to date on Cloudflare I have deployed a [CronJob](./cluster/apps/networking/cloudflare-ddns) in my cluster. This periodically checks and updates the `A` record `ipv4.domain.tld`. I followed the example of [onedr0p/home-ops](https://github.com/onedr0p/home-ops/tree/main/cluster/apps/networking/cloudflare-ddns) as well as docs online. I'll probably write a blog post about it! `external-dns` is pure magic!

---

## ⚡ Network Attached Storage

Right now, I'm running my NAS off an old Dell PowerEdge R510 12 bay server. It's a little power hungry! I run TrueNAS, and have a RAIDz10 setup.

Currently, my raw storage capacity is about 56TB. However, I only get about half that since I'm doing RAIDz10. I have a striped set of 6 mirrored pairs. Effectively I can lose up to half my drives, but if I lose both drives in any pair, I'm having a bad day.

To offset this, I back up the NAS once a week to Google Drive (Google Workspaces), where I pay for an Enterprise account, which gets me a large amount of storage with the caveat of only being able to upload about 750GB a day. Since I completed a main backup of my NAS a long time ago, I only do incremental backups now.

I'm slowly replacing all the 2TB drives, which came with my NAS, as they fail or I catch a sale on nicer WD Easy Store drives that I shuck. These are usually white label WD Red drives, or sometimes even better, HGST drives. My plan is to slowly upgrade everything to 12TB drives.

My NAS can be accessed via NFS over my home network.

---

## 🔧 Hardware

| Device                    | Count | OS Disk Size     | Data Disk Size                 | Ram   | Operating System   | Purpose                        |
|---------------------------|-------|------------------|--------------------------------|-------|--------------------|--------------------------------|
| Poweredge R720            | 1     | 500GB SATA       | N/A                            | 128GB | Ubuntu 20.04       | Kubernetes (k3s) Node          |
| Intel NUC8i7HVK1          | 1     | 256GB NVMe       | N/A                            | 16GB  | Ubuntu 20.04       | Kubernetes (k3s) Node          |
| UCS C220 M3               | 1     | 2x500GB SAS      | N/A                            | 96GB  | Ubuntu 20.04       | Kubernetes (k3s) Node          |
| PowerEdge R510            | 1     | 32GB Flash Drive | 2x12TB, 2x8TB, 8x2TB  RAIDz10  | 16GB  | TrueNAS 12.0       | Network Attached Storage (NAS) |

---

## 🤝 Graditude and Thanks

Thanks to all the people who donate their time to the [Kubernetes @Home](https://github.com/k8s-at-home/) community. A lot of inspiration for my cluster came from the people that have shared their clusters over at [awesome-home-kubernetes](https://github.com/k8s-at-home/awesome-home-kubernetes).

---

## 📜 Changelog

See [commit history](https://github.com/parsec/home-cluster/commits/main)

---

## 🔏 License

See [LICENSE](./LICENSE)
