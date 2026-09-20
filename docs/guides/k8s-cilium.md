export QPS=10
export BURST=15


helm upgrade cilium cilium/cilium --version 1.20.0 \
   --namespace kube-system \
   --reuse-values \
   --set l2announcements.enabled=true \
   --set k8sClientRateLimit.qps={QPS} \
   --set k8sClientRateLimit.burst={BURST} \
   --set enable-lb-ipam=true
   --set kubeProxyReplacement=true

## API Gateway

https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/