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
kubectl -n <NAMESPACE> get events --sort-by='.metadata.creationTimestamp'  
```  

### List ETCD Keys  
```bash
docker exec -ti etcd etcdctl get / --prefix --keys-only  
```  

### LSOF show listening TCP sockets  
```bash
lsof -nP -iTCP -sTCP:LISTEN  
```  


### Extract Kube-API Heap  
Edit the cluster kube-serverapi config file to enable the profiling.  
```yaml
kubeApi:
  extraArgs:
    profiling: "true"
```  

### Get kubeconfig file from controlplane node  
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

### Extract certificate from the above generated kubeconfig file
```bash
echo -n $(cat kubeconfig_admin.yaml | grep client-certificate-data | awk '{print $2}') | base64 -d > cert.pem
echo -n $(cat kubeconfig_admin.yaml | grep client-key-data | awk '{print $2}') | base64 -d > key.pem
```

### Extract Heap file
```bash
curl -k --cacert /etc/kubernetes/ssl/kube-ca.pem --key key.pem --cert cert.pem https://localhost:6443/debug/pprof/heap -o heap
```  

### Test SSL Encryption
```bash
docker run --rm -ti  drwetter/testssl.sh <domain.org>:<sslPort>
```
