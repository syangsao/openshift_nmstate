# openshift_nmstate — br-ex → NMState migration (luke)

Migrating the `br-ex` external bridge on the **luke** OCP 4.20 cluster from
`configure-ovs.sh` to a declarative NMState configuration, per the
[OCP 4.20 post-installation guide](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/installing_on_bare_metal/bare-metal-post-installation-configuration#migrating-br-ex-bridge-nmstate_bare-metal-postinstallation-configuration).

## Pages (GitHub Pages)

| Page | URL |
|---|---|
| Overview & current state | https://syangsao.github.io/openshift_nmstate/index.html |
| NMState manifests | https://syangsao.github.io/openshift_nmstate/nmstate.html |
| Migration procedure | https://syangsao.github.io/openshift_nmstate/migration.html |

## Layout

```
index.html                     overview: current state, plan, warnings
nmstate.html                   manifest documentation
migration.html                 step-by-step procedure + verification + rollback
style.css                      shared styling
nmstate/
  control01-br-ex.yaml         NMState config for control01 (bond0 -> VLAN 40 -> br-ex)
  control02-br-ex.yaml         NMState config for control02 (same topology, .27 IP)
  arbiter-br-ex.yaml           NMState config for arbiter (ens192 -> br-ex)
machineconfigs/
  control01-br-ex-nmstate.yaml MachineConfig embedding control01-br-ex.yaml
                                -> /etc/nmstate/openshift/control01.yml
  control02-br-ex-nmstate.yaml MachineConfig embedding control02-br-ex.yaml
                                -> /etc/nmstate/openshift/control02.yml
  arbiter-br-ex-nmstate.yaml   MachineConfig embedding arbiter-br-ex.yaml
                                -> /etc/nmstate/openshift/cluster.yml
  force-reboot.yaml            bare MCs (master/worker/arbiter) to trigger node reboots
```

## Current state (verified 2026-09-21)

| Node | Role | br-ex uplink | br-ex IPs | MTU |
|---|---|---|---|---|
| control01.syangsao.net | master+worker | bond0 (802.3ad: eno1,eno2) → VLAN 40 | 192.168.40.26/24 + 169.254.0.2/17 | 1500 |
| control02.syangsao.net | master+worker | bond0 (802.3ad: eno1,eno2) → VLAN 40 | 192.168.40.27/24 + 169.254.0.2/17 (+ .28/.29 /32) | 1500 |
| arbiter.syangsao.net | arbiter | ens192 (single NIC) | 192.168.40.25/24 + 169.254.0.2/17 | 1500 |

Bond: 802.3ad, LACP slow, miimon 100, xmit_hash_policy layer2. Gateway
192.168.40.1 (metric 150, OVN-managed). VLANs 60/80/90 on bond0 carry VM data
(br-vmdata) and are out of scope.

## Key rules in the NMState files

- Explicit `mtu: 1500` on every interface (Red Hat requirement).
- OVN-Kubernetes masquerade address `169.254.0.2/17` preserved.
- OVN patch port `patch-br-ex_<node>-to-br-int` omitted (created at runtime).
- Default route not re-declared (OVN owns it).

## Warnings

- **One-way door:** once applied, `configure-ovs.sh` skips on every boot and you
  cannot migrate back to the shell-script bridge.
- **IPs are immutable** post-installation per Red Hat support policy; this
  migration preserves all existing addresses.
- Do it in a maintenance window with iDRAC/iLO console access available.
