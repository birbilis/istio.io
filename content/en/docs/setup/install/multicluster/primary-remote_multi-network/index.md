---
title: Install Primary-Remote on different networks
description: Install an Istio mesh across primary and remote clusters on different networks.
weight: 40
icon: setup
keywords: [kubernetes,multicluster]
test: yes
owner: istio/wg-environments-maintainers
---
Follow this guide to install the Istio control plane on `cluster1` (the
{{< gloss >}}primary cluster{{< /gloss >}}) and configure `cluster2` (the
{{< gloss >}}remote cluster{{< /gloss >}}) to use the control plane in
`cluster1`. Cluster `cluster1` is on the `network1` network, while `cluster2`
is on the `network2` network. This means there is no direct connectivity
between pods across cluster boundaries.

Before proceeding, be sure to complete the steps under
[before you begin](/docs/setup/install/multicluster/before-you-begin).

In this configuration, cluster `cluster1` will observe the API Servers in
both clusters for endpoints. In this way, the control plane will be able to
provide service discovery for workloads in both clusters.

Service workloads across cluster boundaries communicate indirectly, via
dedicated gateways for [east-west](https://en.wikipedia.org/wiki/East-west_traffic)
traffic. The gateway in each cluster must be reachable from the other cluster.

Services in `cluster2` will reach the control plane in `cluster1` via the
same east-west gateway.

{{< image width="75%"
    link="arch.svg"
    caption="Primary and remote clusters on separate networks"
    >}}

{{< tip >}}
Today, the remote profile will install an istiod server in the remote
cluster which will be used for CA and webhook injection for workloads
in that cluster. Service discovery, however, will be directed to the
control plane in the primary cluster.

Future releases will remove the need for having an istiod in the
remote cluster altogether. Stay tuned!
{{< /tip >}}

## Set the default network for `cluster1`

If the istio-system namespace is already created, we need to set the cluster's network there:

{{< text bash >}}
$ kubectl --context="${CTX_CLUSTER1}" get namespace istio-system && \
  kubectl --context="${CTX_CLUSTER1}" label namespace istio-system topology.istio.io/network=network1
{{< /text >}}

## Configure `cluster1` as a primary

Create the Istio configuration for `cluster1`:

{{< text bash >}}
$ cat <<EOF > cluster1.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster1
      network: network1
EOF
{{< /text >}}

Apply the configuration to `cluster1`:

{{< text bash >}}
$ istioctl install --context="${CTX_CLUSTER1}" -f cluster1.yaml
{{< /text >}}

## Install the east-west gateway in `cluster1`

Install a gateway in `cluster1` that is dedicated to east-west traffic. By
default, this gateway will be public on the Internet. Production systems may
require additional access restrictions (e.g. via firewall rules) to prevent
external attacks. Check with your cloud vendor to see what options are
available.

{{< text bash >}}
$ MESH=mesh1 CLUSTER=cluster1 NETWORK=network1 \
    @samples/multicluster/gen-eastwest-gateway.sh@ | \
    istioctl manifest generate -f - | \
    kubectl apply --context="${CTX_CLUSTER1}" -f -
{{< /text >}}

Wait for the east-west gateway to be assigned an external IP address:

{{< text bash >}}
$ kubectl --context="${CTX_CLUSTER1}" get svc istio-eastwestgateway -n istio-system
NAME                    TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)   AGE
istio-eastwestgateway   LoadBalancer   10.80.6.124   34.75.71.237   ...       51s
{{< /text >}}

## Expose the control plane in `cluster1`

Before we can install on `cluster2`, we need to first expose the control plane in
`cluster1` so that services in `cluster2` will be able to access service discovery:

{{< text bash >}}
$ kubectl apply --context="${CTX_CLUSTER1}" -f \
    @samples/multicluster/expose-istiod.yaml@
{{< /text >}}

## Expose services in `cluster1`

Since the clusters are on separate networks, we also need to expose all user
services (*.local) on the east-west gateway in both clusters. While this
gateway is public on the Internet, services behind it can only be accessed by
services with a trusted mTLS certificate and workload ID, just as if they were
on the same network.

{{< text bash >}}
$ kubectl --context="${CTX_CLUSTER1}" apply -n istio-system -f \
    @samples/multicluster/expose-services.yaml@
{{< /text >}}

## Set the default network for `cluster2`

If the istio-system namespace is already created, we need to set the cluster's network there:

{{< text bash >}}
$ kubectl --context="${CTX_CLUSTER2}" get namespace istio-system && \
  kubectl --context="${CTX_CLUSTER2}" label namespace istio-system topology.istio.io/network=network2
{{< /text >}}

## Enable API Server Access to `cluster2`

Before we can configure the remote cluster, we first have to give the control
plane in `cluster1` access to the API Server in `cluster2`. This will do the
following:

- Enables the control plane to authenticate connection requests from
  workloads running in `cluster2`. Without API Server access, the control
  plane will reject the requests.

- Enables discovery of service endpoints running in `cluster2`.

To provide API Server access to `cluster2`, we generate a remote secret and
apply it to `cluster1`:

{{< text bash >}}
$ istioctl x create-remote-secret \
    --context="${CTX_CLUSTER2}" \
    --name=cluster2 | \
    kubectl apply -f - --context="${CTX_CLUSTER1}"
{{< /text >}}

## Configure `cluster2` as a remote

Save the address of `cluster1`’s east-west gateway.

{{< text bash >}}
$ export DISCOVERY_ADDRESS=$(kubectl \
    --context="${CTX_CLUSTER1}" \
    -n istio-system get svc istio-eastwestgateway \
    -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
{{< /text >}}

Now create a remote configuration on `cluster2`.

{{< text bash >}}
$ cat <<EOF > cluster2.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: remote
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster2
      network: network2
      remotePilotAddress: ${DISCOVERY_ADDRESS}
EOF
{{< /text >}}

Apply the configuration to `cluster2`:

{{< text bash >}}
$ istioctl install --context="${CTX_CLUSTER2}" -f cluster2.yaml
{{< /text >}}

## Install the east-west gateway in `cluster2`

As we did with `cluster1` above, install a gateway in `cluster2` that is dedicated
to east-west traffic and expose user services.

{{< text bash >}}
$ MESH=mesh1 CLUSTER=cluster2 NETWORK=network2 \
    @samples/multicluster/gen-eastwest-gateway.sh@ | \
    istioctl manifest generate -f - | \
    kubectl apply --context="${CTX_CLUSTER2}" -f -
{{< /text >}}

Wait for the east-west gateway to be assigned an external IP address:

{{< text bash >}}
$ kubectl --context="${CTX_CLUSTER2}" get svc istio-eastwestgateway -n istio-system
NAME                    TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)   AGE
istio-eastwestgateway   LoadBalancer   10.0.12.121   34.122.91.98   ...       51s
{{< /text >}}

## Expose services in `cluster2`

As we did with `cluster1` above, expose services via the east-west gateway.

{{< text bash >}}
$ kubectl --context="${CTX_CLUSTER2}" apply -n istio-system -f \
    @samples/multicluster/expose-services.yaml@
{{< /text >}}

**Congratulations!** You successfully installed an Istio mesh across primary
and remote clusters on different networks!

## Next Steps

You can now [verify the installation](/docs/setup/install/multicluster/verify).