
## This repo is some notes about setting up camunda platform 8 (helm-chart) with istio

#### What are the files?

1. lb.yaml - MetalLB LoadBalancer, necessary for me getting LoadBalancer service type in a local kind installation.  Heres a link to set that up : https://kind.sigs.k8s.io/docs/user/loadbalancer/ My IP Address will be different from yours.
2. istio_crs.yaml - A collection of Virtual Services and Gateways, this is a config I started with but not necessarily what is working.
3. working_gateway.yaml - output of `kubectl get gateway -o yaml` to see what the running config is.
4. working_virtual_services.yaml - output of `kubectl get gateway -o yaml` to see what running config is.
