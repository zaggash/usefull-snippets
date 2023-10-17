# Kubernetes 

## Kubectl CheatSheet  

### List ALL ressources from a namespace  
```bash
kubectl api-resources --verbs=list --namespaced -o name \
  | xargs -n 1 kubectl get --show-kind --ignore-not-found [-l <LABEL>=<VALUE>] -n <NAMESPACE>  
```  

### Decrypt secrets inline  
```bash
kubectl -n <NAMESPACE> get secrets <SECRET> -o json | jq '.data | map_values(@base64d)'  
```  

### Order kube events by date  
```bash
kubectl -n <NAMESPACE> get events --sort-by='.lastTimestamp'  
```  

## Restore RKE cluster_recovery.yml and cluster_recovery.rkestate files.
JQ and YQ are required

```
#!/bin/bash
echo "Building cluster_recovery.yml..."
echo "Working on Nodes..."
echo 'nodes:' > cluster_recovery.yml
kubectl --kubeconfig kube_config_cluster.yml -n kube-system get configmap full-cluster-state -o json | jq -r .data.\"full-cluster-state\" | jq -r .desiredState.rkeConfig.nodes | yq -P | sed 's/^/  /' | \
sed -e 's/internalAddress/internal_address/g' | \
sed -e 's/hostnameOverride/hostname_override/g' | \
sed -e 's/sshKeyPath/ssh_key_path/g' >> cluster_recovery.yml
echo "" >> cluster_recovery.yml

echo "Working on services..."
echo 'services:' >> cluster_recovery.yml
kubectl --kubeconfig kube_config_cluster.yml -n kube-system get configmap full-cluster-state -o json | jq -r .data.\"full-cluster-state\" | jq -r .desiredState.rkeConfig.services | yq -P | sed 's/^/  /' >> cluster_recovery.yml
echo "" >> cluster_recovery.yml

echo "Working on network..."
echo 'network:' >> cluster_recovery.yml
kubectl --kubeconfig kube_config_cluster.yml -n kube-system get configmap full-cluster-state -o json | jq -r .data.\"full-cluster-state\" | jq -r .desiredState.rkeConfig.network | yq -P | sed 's/^/  /' >> cluster_recovery.yml
echo "" >> cluster_recovery.yml

echo "Working on authentication..."
echo 'authentication:' >> cluster_recovery.yml
kubectl --kubeconfig kube_config_cluster.yml -n kube-system get configmap full-cluster-state -o json | jq -r .data.\"full-cluster-state\" | jq -r .desiredState.rkeConfig.authentication | yq -P | sed 's/^/  /' >> cluster_recovery.yml
echo "" >> cluster_recovery.yml

echo "Working on systemImages..."
echo 'system_images:' >> cluster_recovery.yml
kubectl --kubeconfig kube_config_cluster.yml -n kube-system get configmap full-cluster-state -o json | jq -r .data.\"full-cluster-state\" | jq -r .desiredState.rkeConfig.systemImages | yq -P | sed 's/^/  /' >> cluster_recovery.yml
echo "" >> cluster_recovery.yml
```

```
kubectl --kubeconfig kube_config_cluster.yml -n kube-system get configmap full-cluster-state -o json | jq -r .data.\"full-cluster-state\" | jq -r . > cluster_recovery.rkestate
```

## Etcd CheatSheet

### List ETCD Keys  
```bash
docker exec -ti etcd etcdctl get / --prefix --keys-only  
```  

## Extract Kube-API Heap
### 1. Edit ApiServer config
Edit the cluster kube-serverapi config file to enable the profiling.
```yaml
kubeApi:
  extraArgs:
    profiling: "true"
```

### 2. Get kubeconfig file from controlplane node
```bash
docker run --rm --net=host \
  -v $(docker inspect kubelet \
      --format '{{ range .Mounts }}{{ if eq .Destination "/etc/kubernetes" }}{{ .Source }}{{ end }}{{ end }}')/ssl:/etc/kubernetes/ssl:ro \
  --entrypoint bash $(docker inspect \
      $(docker images -q --filter=label=org.opencontainers.image.source=https://github.com/rancher/hyperkube.git) \
      --format='{{index .RepoTags 0}}' | tail -1) \
  -c 'kubectl --kubeconfig /etc/kubernetes/ssl/kubecfg-kube-node.yaml get configmap -n kube-system full-cluster-state -o json |\
      jq -r .data.\"full-cluster-state\" |\
      jq -r .currentState.certificatesBundle.\"kube-admin\".config |\
      sed -e "/^[[:space:]]*server:/ s_:.*_: \"https://127.0.0.1:6443\"_"' \
  > kubeconfig_admin.yaml
```

### 3. Extract certificate from the above generated kubeconfig file
```bash
echo -n $(cat kubeconfig_admin.yaml | grep client-certificate-data | awk '{print $2}') | base64 -d > cert.pem
echo -n $(cat kubeconfig_admin.yaml | grep client-key-data | awk '{print $2}') | base64 -d > key.pem
```

### 4. Extract Heap file
```bash
curl -k --cacert /etc/kubernetes/ssl/kube-ca.pem --key key.pem --cert cert.pem https://localhost:6443/debug/pprof/heap -o heap
```  

## Extract Rancher Heap Dump
```
for time in {1..5}
do
  for pod in $(kubectl -n cattle-system get pods --no-headers -l app=rancher | cut -d ' ' -f1)
  do
    echo "Getting heap-$time for $pod"
    kubectl -n cattle-system exec "$pod" -c rancher -- curl -s http://localhost:6060/debug/pprof/heap -o heap;
    kubectl -n cattle-system cp "$pod":heap -c rancher "$pod-heap-$time";
    echo "Saved heap $pod-heap-$time"
  done
  sleep 3
done
echo ".........."
echo "[FINISHED]"
```

# Linux/Bash  

## Networking  

### LSOF show listening TCP sockets  
```bash
lsof -nP -iTCP -sTCP:LISTEN  
```  

### Test SSL Encryption  
```bash
docker run --rm -ti  drwetter/testssl.sh <domain.org>:<sslPort>
```  
