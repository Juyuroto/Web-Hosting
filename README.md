# Architecture VPS / Kubernetes — Homelab

> **Objectif :** Héberger un site internet et stocker des données via Kubernetes sur un serveur physique, derrière un routeur PFsense.

---

## Architecture réseau

```
Internet (IP publique)
        │
        ▼
┌───────────────────────┐
│      PFsense          │
│  ┌─────────────────┐  │
│  │  WAN            │  │  ← IP publique FAI
│  │  LAN 1 (Admin)  │  │  ← 192.168.10.0/24
│  │  LAN 2 (Servers)│  │  ← 192.168.11.0/24
│  │  NAT 80/443 ──► │  │  ← Redirige vers MetalLB
│  └─────────────────┘  │
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│    Switch D-Link      │
│  ┌──────────────────┐ │
│  │ VLAN 10 (Admin)  │ │
│  │ VLAN 11 (Servers)│ │
│  └──────────────────┘ │
└───────┬───────┬───────┘
        │       │
        ▼       ▼
┌──────────┐  ┌──────────────────────────────────────┐
│ PC Admin │  │         Serveur Proxmox VE           │
│ Debian   │  │         192.168.11.5                 │
│ .10.x    │  │                                      │
└──────────┘  │  ┌───────────┐  ┌───────────────┐    │
              │  │ k8s-master│  │  k8s-worker-1 │    │
              │  │.11.10     │  │  .11.11       │    │
              │  └───────────┘  └───────────────┘    │
              │  ┌───────────┐  ┌───────────────┐    │
              │  │k8s-worker2│  │    storage    │    │
              │  │.11.12     │  │  .11.20       │    │
              │  └───────────┘  └───────────────┘    │
              └──────────────────────────────────────┘
```

---

## Stack Kubernetes

```
                ┌──────────────────────────────────┐
 Internet ──►   │    Ingress Nginx (MetalLB)       │  ← 192.168.11.100
                │    + Cert-Manager (Let's Encrypt)│
                └──────────────┬───────────────────┘
                               │
               ┌───────────────┼───────────────┐
               ▼               ▼               ▼
        ┌────────────┐  ┌────────────┐  ┌────────────┐
        │  Website   │  │  Backend   │  │  Database  │
        │ Deployment │  │ Deployment │  │StatefulSet │
        └────────────┘  └────────────┘  └──────┬─────┘
                                               │
                                        ┌──────▼──────┐
                                        │  Longhorn   │
                                        │  (Stockage  │
                                        │  persistant)│
                                        └─────────────┘

        ─────────────── Monitoring ────────────────────
                Prometheus + Grafana + Alertmanager
```

---

## Plan des VMs Proxmox

| VM | Rôle | vCPU | RAM | Disque | IP |
|---|---|---|---|---|---|
| `k8s-master` | Control plane Kubernetes | 2 | 4 Go | 40 Go | 192.168.11.10 |
| `k8s-worker-1` | Worker — site web & apps | 4 | 8 Go | 80 Go | 192.168.11.11 |
| `k8s-worker-2` | Worker — données & BDD | 4 | 8 Go | 80 Go | 192.168.11.12 |
| `storage` | Longhorn / NFS | 2 | 4 Go | 500 Go+ | 192.168.11.20 |

---

## Plan d'adressage réseau

| Réseau | VLAN | Plage IP | Usage |
|---|---|---|---|
| Admin | VLAN 10 | 192.168.10.0/24 | PC admin, gestion Proxmox |
| Servers | VLAN 11 | 192.168.11.0/24 | VMs Kubernetes, storage |
| MetalLB pool | — | 192.168.11.100–110 | IPs LoadBalancer K8s |
| Proxmox | VLAN 11 | 192.168.11.5 | Interface web Proxmox |

---

## Points importants

- **Nom de domaine** : il faut un DNS pointant ton IP publique → tondomaine.fr (ou DDNS si IP dynamique)
- **Sauvegardes Proxmox** : configurer des snapshots automatiques des VMs
- **Monitoring** : Prometheus + Grafana dès le départ pour surveiller les ressources
- **Sécurité** : le master K8s ne doit jamais être exposé directement sur le WAN
- **Firewall inter-VLAN** : PFsense doit bloquer le VLAN Servers → VLAN Admin par défaut