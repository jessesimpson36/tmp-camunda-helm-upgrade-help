
## This repo is some notes about setting up camunda platform 8 (helm-chart) with istio

#### What are the files?

1. lb.yaml - MetalLB LoadBalancer, necessary for me getting LoadBalancer service type in a local kind installation.  Heres a link to set that up : https://kind.sigs.k8s.io/docs/user/loadbalancer/ My IP Address will be different from yours.
2. istio_crs.yaml - A collection of Virtual Services and Gateways, this is a config I started with but not necessarily what is working.
3. working_gateway.yaml - output of `kubectl get gateway -o yaml` to see what the running config is.
4. working_virtual_services.yaml - output of `kubectl get gateway -o yaml` to see what running config is.

## What links do I need to help debug my setup of istio?

https://istio.io/latest/docs/ops/common-problems/network-issues/
https://istio.io/latest/docs/reference/config/networking/virtual-service/
https://istio.io/latest/docs/reference/config/networking/gateway/


## What strange issues did you have to look into?

It's important to have 1 Gateway and multiple VirtualServices pointing to the 1 gateway if you use the same TLS certificate for all of your virtual services.

HTTP2 Connection re-use will result in 404's on all but one camunda webapp otherwise.

## What other things helped you?

The istioctl command line option was a big help in debugging. Also ksniff.

Examples:
```
istioctl proxy-config routes --verbose  -n istio-system istio-ingressgateway-767b5dd74c-blwtm
```
This command will print processed routes. So "Did my HTTPS routes even read the config?!"

```
istioctl analyze
```
This command will look at your running cluster and determine if there are common mistakes you're falling victim to.


Also simply knowing to delete application pods so that they come back up with the istio sidecar injected. This also requires 

```
kubectl label namespace default istio-injection=enabled
```
Before killing/restarting the pods.
