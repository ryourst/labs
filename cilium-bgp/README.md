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
### Install kubectl
```
[[ ! -d "`echo ~`/.local/bin" ]] && mkdir -p "`echo ~`/.local/bin"
kubectl -h &> /dev/null || (echo "Downloading kubectl"; curl -Lo "`echo ~`/.local/bin/kubectl" https://dl.k8s.io/release/v1.24.0/bin/linux/amd64/kubectl )
chmod +x "`echo ~`/.local/bin/"*
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
### Download and Deploy Cilium
```
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/master/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```
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
kubectl apply -f netshoot.yaml
kubectl rollout status ds/netshoot -w
SRC_POD=$(kubectl get pods -o wide | grep "cplane-demo-worker " | awk '{ print($1); }')
DST_IP=$(kubectl get pods -o wide | grep worker3 | awk '{ print($6); }')
kubectl exec -it $SRC_POD -- ping $DST_IP
```

### Notes
https://medium.com/@charled.breteche/kind-cilium-metallb-and-no-kube-proxy-a9fe66ddfad6
https://medium.com/@charled.breteche/kind-cluster-with-cilium-and-no-kube-proxy-c6f4d84b5a9d
https://kind.sigs.k8s.io/docs/user/loadbalancer/

