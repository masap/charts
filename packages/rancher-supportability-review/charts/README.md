# Rancher Support Assistant Chart

The supportability-review-operator provides Support Assistant for Rancher.

# Installation

```
helm install rancher-supportability-review ./ --create-namespace -n sr-operator-system
```

# Node clock check

With `clockCheck.enabled=true`, the chart creates Fleet Bundles that deploy a
DaemonSet to the Rancher local cluster and to the downstream clusters selected
by `clockCheck.clusterSelector`. On every node, it checks every
`clockCheck.interval` that:

- the kernel reports the clock as synchronized by a time daemon such as chronyd,
- the maximum error of the clock that the kernel estimates is within
  `clockCheck.maxError`, and
- the clock is within `clockCheck.maxOffset` of the API server clock.

When the time daemon stops synchronizing the clock, the kernel increases the
maximum error by 0.5 ms every second, and flags the clock as unsynchronized
only when it exceeds 16 seconds, after about 9 hours. `clockCheck.maxError`
(default 1.5s) detects it after about 50 minutes. Stopping chronyd or
systemd-timesyncd does not flag the clock as unsynchronized either, so a
stopped time daemon is reported as `ClockErrorTooLarge` first, as tested with
chronyd on SLES and systemd-timesyncd on Ubuntu 22.04. systemd-timesyncd resets the
maximum error to 0 on every synchronization, so it reaches about 1s between
synchronizations while the clock is fine; keep `clockCheck.maxError` above
that. chronyd sets it to its error estimate, so with chronyd on all nodes it
can be lowered down to 0.5s, which keeps the clocks of any two nodes within 1s,
the clock difference above which etcd warns.

The offset from the API server does not tell which of the two clocks is off.
When the clock of a control plane node is off, the other nodes report
`ClockSkewDetected`, while the control plane node itself may not, because it
can compare its clock with the API server running on it. Its kernel still
reports a stopped time daemon.

A problem is recorded as a Warning event of the node, with the reason
`ClockNotSynchronized`, `ClockErrorTooLarge`, `ClockSkewDetected` or
`ClockCheckFailed`, and the recovery as a Normal event with the reason
`ClockSynchronized`. The events are in the default namespace:

```
kubectl get events -n default --field-selector source=sr-clock-checker
```

The DaemonSet runs the operator image (`image.operator`) without image pull
secrets. With a private registry that requires authentication, the nodes of
every checked cluster, including the local cluster, must be able to pull that
image with their own registry configuration: the Private Registry setting of
the cluster in Rancher, or `registries.yaml` of RKE2/K3s.
`privateRegistry.registrySecret` is not used for the DaemonSet.

The check is disabled by default. Disabling it again or uninstalling the chart
removes the DaemonSet and its namespace (`clockCheck.namespace`) from all
clusters.
