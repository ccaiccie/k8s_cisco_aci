## Helm chart to install CCP ACI service

#### Clone this repo

```
git clone git@github.com:contiv/ccp_aci_service.git

cd ccp_aci_service/k8s/helm
```

#### Install the helm chart that deploys CCP ACI service and its etcd database

First, make sure `helm init` is done and tiller is up and running:

```
$ kubectl get pods -n=kube-system | grep tiller
tiller-deploy-59d854595c-92sww          1/1       Running   10         1d
```

Install the helm chart that deploys CCP ACI service and its etcd database:

```
helm install --name=ccp-test ./
```

#### Check if the helm chart is installed successfully

```
$ helm list
NAME    	REVISION	UPDATED                 	STATUS  	CHART           	NAMESPACE
ccp-test	1       	Fri Mar  9 03:32:53 2018	DEPLOYED	aci-server-0.0.1	default  

$ helm status ccp-test
```

#### Check the CCP ACI server pod and the service

```
$ kubectl get pods
NAME                                                 READY     STATUS    RESTARTS   AGE
ccp-test-aci-server-6d64f959d5-j4qg5                 2/2       Running   0          1m

$ kubectl get svc
NAME                                TYPE        CLUSTER-IP       EXTERNAL-IP      PORT(S)                      AGE
ccp-test-aci-server                 ClusterIP   10.99.93.214     <none>           46802/TCP                    2m

$ kubectl describe service ccp-test-aci-server
Name:              ccp-test-aci-server
Namespace:         default
Labels:            app=aci-server
                   chart=aci-server-0.0.1
                   heritage=Tiller
                   release=ccp-test
Annotations:       <none>
Selector:          app=aci-server,release=ccp-test
Type:              ClusterIP
IP:                10.99.93.214
Port:              http  46802/TCP
TargetPort:        46802/TCP
Endpoints:         172.17.0.17:46802
Session Affinity:  None
Events:            <none>
```

The `ccp-test-aci-server` k8s service above is of type `ClusterIP`.

#### Curl and check the CCP ACI service

`10.99.93.214` below is the `ClusterIP` of the `ccp-test-aci-server` k8s service.

```
$ minikube ssh "curl 10.99.93.214:46802"
{
  "acc-provision": {
    "git_sha1": "e63ecc090cf60f381563b754c170ac09b7edc943", 
    "url": [
      "HTTP POST   /api/v1/acc_provision_create", 
      "HTTP DELETE /api/v1/acc_provision_delete", 
      "HTTP GET    /api/v1/acc_provision_status", 
      "HTTP GET    /"
    ], 
    "version": "1.8.0"
  }
}
```

#### Check the `ccp-test-aci-server` and `ccp-test-aci-server-etcd` containers in the pod and their images

```
$ kubectl describe pod ccp-test-aci-server-6d64f959d5-j4qg5 | grep 'aci-server:\|aci-server-etcd:\|Image:'
  ccp-test-aci-server:
    Image:         containers.cisco.com/contiv/ccp-aci-service:1.0
  ccp-test-aci-server-etcd:
    Image:         k8s.gcr.io/etcd-amd64:3.1.11
```

#### Check the `ccp-test-aci-server-configmap` configmap

```
$ kubectl get configmaps | grep aci
ccp-test-aci-server-configmap     1         1m

$ kubectl describe configmap ccp-test-aci-server-configmap
Name:         ccp-test-aci-server-configmap
Namespace:    default
Labels:       app=aci-server
              chart=aci-server-0.1.0
              heritage=Tiller
              release=ccp-test
Annotations:  <none>

Data
====
aci.conf:
----
[DEFAULT]
DEFAULT_VLAN_MIN = 3130

# DEFAULT_VLAN_MAX's maximum value is 4095
DEFAULT_VLAN_MAX = 3800

# DEFAULT_MULTICAST_RANGE should be a multicast range
DEFAULT_MULTICAST_RANGE = 225.65.0.0/16

DEFAULT_SERVICE_SUBNET = 100.33.0.0/24

# DEFAULT_POD_SUBNET has to end with .1
DEFAULT_POD_SUBNET = 100.44.55.1/16

Events:  <none>
```

#### Check contents of the configmap for aci.conf inside the `ccp-test-aci-server` container

```
$ kubectl exec ccp-test-aci-server-6d64f959d5-j4qg5 -c=ccp-test-aci-server cat /aci.conf
[DEFAULT]
DEFAULT_VLAN_MIN = 3130

# DEFAULT_VLAN_MAX's maximum value is 4095
DEFAULT_VLAN_MAX = 3800

# DEFAULT_MULTICAST_RANGE should be a multicast range
DEFAULT_MULTICAST_RANGE = 225.65.0.0/16

DEFAULT_SERVICE_SUBNET = 100.33.0.0/24

# DEFAULT_POD_SUBNET has to end with .1
DEFAULT_POD_SUBNET = 100.44.55.1/16
```

#### (Optional) Check the stdout logs of the `ccp-test-aci-server` container in the `ccp-test-aci-server-6d64f959d5-j4qg5` pod

```
kubectl logs ccp-test-aci-server-6d64f959d5-j4qg5 -c=ccp-test-aci-server
```

#### (Optional) Check the state in etcd in the `ccp-test-aci-server-etcd` container in the `ccp-test-aci-server-6d64f959d5-j4qg5` pod

```
kubectl exec ccp-test-aci-server-6d64f959d5-j4qg5 -c=ccp-test-aci-server-etcd \
    -- sh -c "ETCDCTL_API=3 etcdctl get --prefix /"
```

#### Delete the helm chart

```
$ helm delete --purge ccp-test
release "ccp-test" deleted

$ helm list
$
```

## Configurations

The following are the configurations for the CCP ACI service and etcd. They can either be passed as command-line arguments using `--set` to `helm install --name=ccp-test ./`, or set in `values.yaml` in this directory:

Parameter | Description | Default value
--- | --- | ---
`CcpAciServer.image.repository` | CCP ACI service's Docker image | `containers.cisco.com/contiv/ccp-aci-service`
`CcpAciServer.image.tag` | Docker tag of CCP ACI service's Docker image | `"1.0"`
`CcpAciServer.image.pullPolicy` | k8s' `pullPolicy` for CCP ACI service's Docker image | `Always`
`CcpAciServer.port` | Port in the container listened by the CCP ACI service | `46802`
`replicaCount` | Number of replicas of the k8s pod | `1`
`service.type` | k8s service type for CCP ACI server | `ClusterIP`
`service.port` | `port` and `targetPort` for the k8s service | `46802`
`etcd.image.repository` | etcd Docker image | `k8s.gcr.io/etcd-amd64`
`etcd.image.tag` | Docker tag of etcd Docker image | `3.1.11`
`etcd.image.pullPolicy` | k8s' `pullPolicy` for etcd Docker image | `IfNotPresent`
`etcd.port` | Port in the container listened by etcd | `2379`
`aciConfigFile` | configmap configurations for `/aci.conf` | refer `values.yaml`
`resources` | k8s CPU and memory resources for the deployment | `{}`
`nodeSelector` | k8s nodeSelector values | `{}`
`tolerations` | k8s tolerations | `[]`
`affinity` | k8s affinity values | `{}`

The above configurations can be passed as command-line arguments using `--set` to `helm install --name=ccp-test ./` as follows:

```
helm install --name=ccp-test ./ \
    --set CcpAciServer.image.repository=containers.cisco.com/contiv/ccp-aci-service \
    --set CcpAciServer.image.tag="1.0" \
    --set service.type=ClusterIP \
    --set etcd.image.repository=k8s.gcr.io/etcd-amd64 \
    --set etcd.image.tag=3.1.11

$ helm list
NAME    	REVISION	UPDATED                 	STATUS  	CHART           	NAMESPACE
ccp-test	1       	Fri Mar  9 04:09:47 2018	DEPLOYED	aci-server-0.0.1	default  

$ helm status ccp-test

$ helm delete --purge ccp-test
release "ccp-test" deleted

$ helm list
$
```
