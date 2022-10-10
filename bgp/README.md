### Install Docker
https://www.docker.com/

### Install Kind
```
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.16.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
# edit cluster.yaml if needed
kind create cluster --config=./cluster.yaml
```
### Install containerlab
```
bash -c "$(curl -sL https://get.containerlab.dev)" -- -v 0.31.1
```
### Deploy lab
```
containerlab -t topo.yaml deploy
# test
docker exec -it clab-bgp-cplane-demo-router0 vtysh -c 'show bgp ipv4 summary wide'
docker exec -it clab-bgp-cplane-demo-tor0 vtysh -c 'show bgp ipv4 summary wide'
docker exec -it clab-bgp-cplane-demo-tor1 vtysh -c 'show bgp ipv4 summary wide'
```
### Deploy Cilium
```
cilium install \
    --helm-set ipam.mode=kubernetes \
    --helm-set tunnel=disabled \
    --helm-set ipv4NativeRoutingCIDR="10.0.0.0/8" \
    --helm-set bgpControlPlane.enabled=true \
    --helm-set k8s.requireIPv4PodCIDR=true
cilium status
cilium config view | grep enable-bgp
```
### Configure BGP peering
```
kubectl apply -f cilium-bgp-peering-policies.yaml
```
### Verify
```
docker exec -it clab-bgp-cplane-demo-tor0 vtysh -c 'show bgp ipv4 summary wide'
docker exec -it clab-bgp-cplane-demo-tor1 vtysh -c 'show bgp ipv4 summary wide'
docker exec -it clab-bgp-cplane-demo-tor0 vtysh -c 'show ip bgp'
docker exec -it clab-bgp-cplane-demo-tor1 vtysh -c 'show ip bgp'
# deploy  test app
kubectl apply -f netshoot-ds.yaml
kubectl rollout status ds/netshoot -w
SRC_POD=$(kubectl get pods -o wide | grep "cplane-demo-worker " | awk '{ print($1); }')
DST_IP=$(kubectl get pods -o wide | grep worker3 | awk '{ print($6); }')
kubectl exec -it $SRC_POD -- ping $DST_IP
```

### Notes
https://medium.com/@charled.breteche/kind-cilium-metallb-and-no-kube-proxy-a9fe66ddfad6
https://medium.com/@charled.breteche/kind-cluster-with-cilium-and-no-kube-proxy-c6f4d84b5a9d
https://kind.sigs.k8s.io/docs/user/loadbalancer/

