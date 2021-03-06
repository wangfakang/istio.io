---
title: Istio Operator Install
description: Instructions to install Istio in a Kubernetes cluster using the Istio operator.
weight: 25
keywords: [kubernetes, operator]
aliases:
owner: istio/wg-environments-maintainers
test: no
---

Instead of manually installing, upgrading, and uninstalling Istio in a production environment,
you can instead let the Istio [operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
manage the installation for you.
This relieves you of the burden of managing different `istioctl` versions.
Simply update the operator {{<gloss CRDs>}}custom resource (CR){{</gloss>}} and the
operator controller will apply the corresponding configuration changes for you.

The same [`IstioOperator` API](/docs/reference/config/istio.operator.v1alpha1/) is used
to install Istio with the operator as when using the [istioctl install instructions](/docs/setup/install/istioctl).
In both cases, configuration is validated against a schema and the same correctness
checks are performed.

{{< warning >}}
Using an operator does have a security implication.
With the `istioctl install` command, the operation will run in the admin user’s security context,
whereas with an operator, an in-cluster pod will run the operation in its security context.
To avoid a vulnerability, ensure that the operator deployment is sufficiently secured.
{{< /warning >}}

## Prerequisites

1. Perform any necessary [platform-specific setup](/docs/setup/platform-setup/).

1. Check the [Requirements for Pods and Services](/docs/ops/deployment/requirements/).

1. Install the [{{< istioctl >}} command](/docs/ops/diagnostic-tools/istioctl/).

1. Deploy the Istio operator:

    {{< text bash >}}
    $ istioctl operator init
    {{< /text >}}

    This command runs the operator by creating the following resources in the `istio-operator` namespace:

    - The operator custom resource definition
    - The operator controller deployment
    - A service to access operator metrics
    - Necessary Istio operator RBAC rules

    You can configure which namespace the operator controller is installed in, the namespace(s) the operator watches, the installed Istio image sources and versions, and more. For example, you can pass one or more namespaces to watch using the `--watchedNamespaces` flag:

    {{< text bash >}}
    $ istioctl operator init --watchedNamespaces=istio-namespace1,istio-namespace2
    {{< /text >}}

    See the [`istioctl operator init` command reference](/docs/reference/commands/istioctl/#istioctl-operator-init) for details.

    {{< tip >}}
    You can alternatively deploy the operator using Helm:

    {{< text bash >}}
    $ helm template manifests/charts/istio-operator/ \
      --set hub=docker.io/istio \
      --set tag={{< istio_full_version >}} \
      --set operatorNamespace=istio-operator \
      --set watchedNamespaces=istio-system | kubectl apply -f -
    {{< /text >}}

    Note that you need to [download the Istio release](/docs/setup/getting-started/#download)
    to run the above command.
    {{< /tip >}}

## Install

To install the Istio `demo` [configuration profile](/docs/setup/additional-setup/config-profiles/)
using the operator, run the following command:

{{< text bash >}}
$ kubectl create ns istio-system
$ kubectl apply -f - <<EOF
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: example-istiocontrolplane
spec:
  profile: demo
EOF
{{< /text >}}

The controller will detect the `IstioOperator` resource and then install the Istio
components corresponding to the specified (`demo`) configuration.

{{< tip >}}
The Istio operator controller begins the process of installing Istio within 90 seconds of
the creation of the `IstioOperator` resource. The Istio installation completes within 120
seconds.
{{< /tip >}}

You can confirm the Istio control plane services have been deployed with the following commands:

{{< text bash >}}
$ kubectl get svc -n istio-system
NAME                        TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                      AGE
grafana                     ClusterIP      10.104.1.213     <none>        3000/TCP                                                                     13s
istio-egressgateway         ClusterIP      10.103.243.113   <none>        80/TCP,443/TCP,15443/TCP                                                     17s
istio-ingressgateway        LoadBalancer   10.101.204.227   <pending>     15020:31077/TCP,80:30689/TCP,443:32419/TCP,31400:31411/TCP,15443:30176/TCP   17s
istiod                      ClusterIP      10.96.237.249    <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP,53/UDP,853/TCP                         30s
jaeger-agent                ClusterIP      None             <none>        5775/UDP,6831/UDP,6832/UDP                                                   13s
jaeger-collector            ClusterIP      10.109.244.165   <none>        14267/TCP,14268/TCP,14250/TCP                                                13s
jaeger-collector-headless   ClusterIP      None             <none>        14250/TCP                                                                    13s
jaeger-query                ClusterIP      10.105.128.63    <none>        16686/TCP                                                                    13s
kiali                       ClusterIP      10.102.172.30    <none>        20001/TCP                                                                    13s
prometheus                  ClusterIP      10.102.190.194   <none>        9090/TCP                                                                     13s
tracing                     ClusterIP      10.110.231.250   <none>        80/TCP                                                                       13s
zipkin                      ClusterIP      10.96.63.117     <none>        9411/TCP                                                                     13s
{{< /text >}}

{{< text bash >}}
$ kubectl get pods -n istio-system
NAME                                   READY   STATUS    RESTARTS   AGE
grafana-54b54568fc-fk6p5               1/1     Running   0          82s
istio-egressgateway-5444c68db8-9h6dz   1/1     Running   0          87s
istio-ingressgateway-5c68cb968-x7qv9   1/1     Running   0          87s
istio-tracing-9dd6c4f7c-hd9qq          1/1     Running   0          82s
istiod-598984548d-wjq9j                1/1     Running   0          99s
kiali-d45468dc4-4nqdv                  1/1     Running   0          82s
prometheus-6cf46c47fb-tvvkv            2/2     Running   0          82s
{{< /text >}}

## Update

Now, with the controller running, you can change the Istio configuration by editing or replacing
the `IstioOperator` resource. The controller will detect the change and respond by updating
the Istio installation correspondingly.

For example, you can switch the installation to the `default`
profile with the following command:

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: example-istiocontrolplane
spec:
  profile: default
EOF
{{< /text >}}

You can also enable or disable components and modify resource settings.
For example, to enable the `Grafana` component and increase pilot memory requests:

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: example-istiocontrolplane
spec:
  profile: default
  components:
    pilot:
      k8s:
        resources:
          requests:
            memory: 3072Mi
  addonComponents:
    grafana:
      enabled: true
EOF
{{< /text >}}

You can observe the changes that the controller makes in the cluster in response to `IstioOperator` CR updates by
checking the operator controller logs:

{{< text bash >}}
$ kubectl logs -f -n istio-operator $(kubectl get pods -n istio-operator -lname=istio-operator -o jsonpath='{.items[0].metadata.name}')
{{< /text >}}

Refer to the [`IstioOperator` API](/docs/reference/config/istio.operator.v1alpha1/#IstioOperatorSpec)
for the complete set of configuration settings.

## Uninstall

Delete the Istio deployment:

{{< text bash >}}
$ kubectl delete istiooperators.install.istio.io -n istio-system example-istiocontrolplane
{{< /text >}}

Wait until Istio is uninstalled - this may take some time.
Delete the Istio operator:

{{< text bash >}}
$ istioctl operator remove
{{< /text >}}

Or:

{{< text bash >}}
$ kubectl delete ns istio-operator --grace-period=0 --force
{{< /text >}}

Note that deleting the operator before Istio is fully removed may result in leftover Istio resources.
To clean up anything not removed by the operator:

{{< text bash >}}
$ istioctl manifest generate | kubectl delete -f -
$ kubectl delete ns istio-system --grace-period=0 --force
 {{< /text >}}
