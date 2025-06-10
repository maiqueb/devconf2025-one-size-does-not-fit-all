# devconf2025-one-size-does-not-fit-all
Repo with instructions and manifests for the DevConf 2025 talk [Kubernetes Networking: One size doesn't fit all](https://pretalx.devconf.info/devconf-cz-2025/talk/LVKTGW/).

## Requirements
- [kind](https://kind.sigs.k8s.io/)
- [helm](https://helm.sh/)
- container runtime (docker / podman)

## Cluster creation instructions
1. Create a kind cluster (no CNI installed please).
```
kind create cluster --config manifests/empty-kind.yaml
kind export kubeconfig --name ovn-k-helm
```
2. Label the kind nodes
```
for node in $(kind get nodes --name "ovn-k-helm"); do
  kubectl label node "${node}" k8s.ovn.org/zone-name=${node} --overwrite
done
```
3. Clone the ovn-kubernetes repo
```
git clone https://github.com/ovn-kubernetes/ovn-kubernetes.git
```
4. Install OVN-Kubernetes in your kind cluster
```
cd ovn-kubernetes/helm/ovn-kubernetes
helm install . --generate-name -f ./values-single-node-zone.yaml \
    --set k8sAPIServer="https://$(kubectl get pods -n kube-system -l component=kube-apiserver -o jsonpath='{.items[0].status.hostIP}'):6443" \
    --set ovnkube-identity.replicas=$(kubectl get node -l node-role.kubernetes.io/control-plane --no-headers | wc -l) \
    --set global.image.repository=ghcr.io/ovn-kubernetes/ovn-kubernetes/ovn-kube-ubuntu \
    --set global.image.tag=master \
    --set global.enableMultiNetwork="true" \
    --set global.enableNetworkSegmentation="true"
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/refs/heads/master/deployments/multus-daemonset.yml
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multi-networkpolicy/refs/tags/v1.0.1/scheme.yml
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/ipamclaims/v0.4.0-alpha/artifacts/k8s.cni.cncf.io_ipamclaims.yaml

# lets wait for the availability of the ovnkube-node daemonset on all nodes ...
kubectl rollout status daemonset -novn-kubernetes ovnkube-node --timeout 300s
```
5. Install KubeVirt in your kind cluster
```
# install kubevirt
export RELEASE=$(curl https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)
kubectl apply -f "https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-operator.yaml"
kubectl apply -f "https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-cr.yaml"
kubectl -n kubevirt wait kv kubevirt --for condition=Available --timeout=300s 

# install ipam extensions
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.yaml
kubectl wait -ncert-manager deployment cert-manager-webhook \
    --for condition=Available \
    --timeout=60s

kubectl apply -f https://raw.githubusercontent.com/kubevirt/ipam-extensions/main/dist/install.yaml
kubectl wait -nkubevirt-ipam-controller-system deployment kubevirt-ipam-controller-manager \
    --for condition=Available \
    --timeout=60s

# configure additional network binding for primary UDN
kubectl patch -n kubevirt kubevirt kubevirt \
    --type=json \
    --patch '[
        {"op":"add","path":"/spec/configuration/network","value":{}},
        {"op":"add","path":"/spec/configuration/network/binding","value":{"l2bridge":{"domainAttachmentType":"managedTap","migration":{}}}}
    ]'
```

