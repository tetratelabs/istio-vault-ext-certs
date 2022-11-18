# Local Setup

We have provided a full `docker-compose` environment containing two kubernetes clusters (based on [Rancher K3s](https://docs.k3s.io)) and a vault instance. In order to spin up this local demo environment, issue the following commands.

```console
docker-compose up -d
```
```
  [+] Running 8/8

  ⠿ Network demo            Created
  ⠿ Volume "k3s-server-1"   Created
  ⠿ Volume "k3s-server-2"   Created
  ⠿ Container k3s-server-2  Started
  ⠿ Container k3s-server-1  Started
  ⠿ Container vault         Started
  ⠿ Container k3s-agent-1a  Started
  ⠿ Container k3s-agent-2a  Started  
```

Check the cluster status by using the corresponding `kubecfg1.yml` and `kubecfg2.yml` files. These files are created during the bootstrap process of the k3s servers and mounted into the present working directory. Verify if the nodes and pods are correctly started.

```console
kubectl --kubeconfig kubecfg1.yml get nodes -A
kubectl --kubeconfig kubecfg1.yml get pods -A

kubectl --kubeconfig kubecfg2.yml get nodes -A
kubectl --kubeconfig kubecfg2.yml get pods -A
```

```console
  NAME           STATUS   ROLES                  AGE   VERSION
  100fbe327353   Ready    control-plane,master   21s   v1.24.7+k3s1
  d3401f675eaf   Ready    <none>                 18s   v1.24.7+k3s1

  NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
  kube-system   local-path-provisioner-7b7dc8d6f5-92wnv   1/1     Running   0          12s
  kube-system   coredns-b96499967-flvwc                   1/1     Running   0          12s
```

Store the vault and kubernetes API server endpoints in a shell environment variable for further use.

```console
export VAULT_SERVER=http://`docker inspect --format "{{ .NetworkSettings.Networks.demo.IPAddress }}" vault`:8200
export K8S_API_SERVER_1=https://`docker inspect --format "{{ .NetworkSettings.Networks.demo.IPAddress }}" k3s-server-1`:16443
export K8S_API_SERVER_2=https://`docker inspect --format "{{ .NetworkSettings.Networks.demo.IPAddress }}" k3s-server-2`:26443

echo "VAULT_SERVER=$VAULT_SERVER"
echo "K8S_API_SERVER_1=$K8S_API_SERVER_1"
echo "K8S_API_SERVER_2=$K8S_API_SERVER_2"
```

```
  VAULT_SERVER=http://172.18.0.4:8200
  K8S_API_SERVER_1=https://172.18.0.2:16443
  K8S_API_SERVER_2=https://172.18.0.3:26443
```

> **NOTE:** The root token to login to the vault UI is hard-coded to `root` within `docker-compose.yml`.


</br>

## Cleanup

In order to bring down the demo environment, including the named volumes.

```console
docker-compose down -v
```
```
  [+] Running 8/8
  ⠿ Container k3s-agent-1a  Removed
  ⠿ Container k3s-agent-2a  Removed
  ⠿ Container vault         Removed
  ⠿ Container k3s-server-1  Removed
  ⠿ Container k3s-server-2  Removed
  ⠿ Volume k3s-server-1     Removed
  ⠿ Volume k3s-server-2     Removed                            
  ⠿ Network demo            Removed
```
