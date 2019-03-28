# flannel2cilium

Experiments about full migration of Kubernetes cluster from flannel to Cilium.
Currently this repo is just for collecting ideas and small scripts, not a full
solution. But hopefully at some point it will start evolving into a proper set
of instructions or even a tool.

## Problem

The problem which the CNI plugin from flannel to Cilium on the
already running Kubernetes cluster with the smallest disruption possible.

## Idea

The idea is focused around concepts of Node labels and Node selectors.

Let's assume that you have a Kubernetes cluster with flannel as a CNI plugin
and with multiple Deployments, DaemonSets (outside `kube-system` namespace) or
StatefulSets.

The migration should start with the following initial steps:

- Labeling all Nodes with `cni=flannel`.
- Patching all Pods, Deployments, StatefulSets, DaemonSets (not the ones which
are in `kube-system` namespace) with `nodeSelector` - `cni: flannel`.
- Patching the flannel DaemonSet with `nodeSelector` - `cni: flannel`.

After the steps above are done:

- Deploy **patched** Cilium DaemonSet which has a `nodeSelector` - `cni: cilium`.

Then the following steps need to be done for each **n/2+1** worker nodes:

- Setting `cni=none` label.
- Waiting until Pods managed by flannel DaemonSet and Pods managed by any
Deployment, StatefulSet or DaemonSet (outside `kube-system` namespace) are not
scheduled on that node anymore.
- Draining the Node - `kubectl drain --ignore-daemonsets`.
- Logging into the Node (ssh) and executing the following commands:
  - `ip link delete cni0`
  - `ip link delete flannel.1`
- Mark the node again as schedulable - `kubectl uncordon`.

After everything was done with n/2+1 nodes, let's try to evict flannel from the
master node:

- Label the master node with `cni=none`.
- Wait until Pods managed by flannel DaemonSet are not present on the master
  node anymore.
- Login into the master (ssh) and execute the following commands:
  - `ip link delete cni0`
  - `ip link delete flannel.1`

After that's done, it means that we have master node and n/2+1 nodes without
any CNI plugin running and without network interfaces which were previously
used by flannel. Let's try to install Cilium on them:

- Label all the nodes which are labeled with `cni=none` as `cni=cilium`.
- Wait until Pods managed by Cilium DaemonSet appear on all Nodes labeled as
`cni=cilium`.

Then:

- Change `nodeSelector` from `cni: flannel` to `cni: cilium` on all Pods,
Deployments, StatefulSets, DaemonSets (those outside `kube-system` namespace)
- Wait until all of them get scheduled in nodes labeled with `cni=cilium`.

Then for the rest of n/2-1 worker nodes which still have `cni=flannel` labels:

- Set `cni=none` label.
- Wait until Pods managed by flannel DaemonSet are not scheduled on that node
anymore.
- Drain the Node - `kubectl drain --ignore-daemonsets`.
- Log into the Node and execute the following commands:
  - `ip link delete cni0`
  - `ip link delete flannel.1`
- Mark the node again as schedulable - `kubectl uncordon`.
- Label the node with `cni=cilium`.

After that the migration should be done and it should be safe to just:

- Delete the flannel DaemonSet.
- Remove `cni` labels from all nodes.
- Remove `nodeSelector` from Cilium DaemonSet spec.

## Problems

After trying that approach, Services get unaccessable and they need to be
created again. Will also labeling `kube-proxy` and keeping it only with nodes
which preferred CNI (at the moment of migration) improve anything? That's the
open question.
