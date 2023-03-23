# IPMI Metrics on OpenShift

This project deploys the [`ipmi_exporter`][ix] project and arranges to expose those metrics to Prometheus. We configure [`kube-rbac-proxy`][krp] to control access to the metrics endpoint, and we use [`cert-manager`][cm] to automatically provision SSL certificates for `kube-rbac-proxy`.

## Gathering IPMI metrics

We deploy [`ipmi_exporter`][ix] using a [DaemonSet][], which ensures that a pod is spawned on each node in the cluster. In order to access IPMI data, `ipmi_exporter` needs access to `/dev/ipmi0`. Access to host devices under Kubernetes requires running a container in `privileged` mode, so we include RBAC to grant the `ipmi-exporter` ServiceAccount access to the `privileged` and `anyuid` [SecurityContextConstraints][scc].

## Limiting access to the metrics endpoint

The `/metrics` endpoint provided by `ipmi_exporter` performs no authorization checks. In order to limit access to the hardware metrics (e.g., by a malicious process running in another namespace), we use [kube-rbac-proxy][krp]. This provides us with:

1. An encrypted HTTPS endpoint, rather than plaintext HTTP, and
2. Access control configured using Kubernetes RBAC resources

In order to perform the authorization checks, we need a ClusterRole that grants the `ipmi-exporter` ServiceAccount access to the TokeNReview and SubjectAccessReview APIs.

### Internal certificate authority

We want Prometheus to trust the certificate presented by the HTTPS endpoint provided by `kube-rbac-proxy`. We achieve this by bootstrapping an internal certificate authority using a [cert-manager][cm] "selfsigned" issuer. The process looks like:

1. Use the selfsigned issuer to create a new self-signed certificate.
2. Create a new Issuer that signs certificates using the certificate generated in the previous step
3. Generate a certificate for `kube-rbac-proxy` using the new Issuer
4. Convert Prometheus (via the ServiceConfig manifest) to trust the CA certificate generated in the first step.

## Exposing metrics to Prometheus

We create a [ServiceMonitor][] resource to expose the resource to Prometheus. This tells Prometheus at which address and at which path to find the metrics. It also performs some label transformations; in particular, it replaces the `instance` label with the name of the node on which the `ipmi-metrics` pod is running.

## Labelling the namespace

Lastly, we label the namespace in which `ipmi_exporter` is running with the `openshift.io/cluster-monitoring=true` label, which causes the Prometheus operator to discover ServiceMonitor resources in this namespace.

## See also

- Prometheus [`kubernetes_sd_config` documentation](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#kubernetes_sd_config)


[ix]: https://github.com/prometheus-community/ipmi_exporter
[krp]: https://github.com/brancz/kube-rbac-proxy
[servicemonitor]: https://docs.openshift.com/container-platform/4.10/rest_api/monitoring_apis/servicemonitor-monitoring-coreos-com-v1.html
[daemonset]: https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/
[scc]: https://docs.openshift.com/container-platform/4.10/authentication/managing-security-context-constraints.html
[cm]: https://cert-manager.io/
