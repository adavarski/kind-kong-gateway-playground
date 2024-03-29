## Kong API Gateway on KinD
This guide introduces anyone who is interested in doing local hands-on experience with Kong Gateway on KinD cluster

### Dependencies

- [Docker](https://docs.docker.com/engine/install/ubuntu/)
- [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/docs/intro/install/)

### Create k8s cluster (KinD)
```
cd kind/
kind create cluster --name=kong-cluster --config=config.yml

$ kubectl get --raw '/healthz?verbose'
[+]ping ok
[+]log ok
[+]etcd ok
[+]poststarthook/start-kube-apiserver-admission-initializer ok
[+]poststarthook/generic-apiserver-start-informers ok
[+]poststarthook/priority-and-fairness-config-consumer ok
[+]poststarthook/priority-and-fairness-filter ok
[+]poststarthook/storage-object-count-tracker-hook ok
[+]poststarthook/start-apiextensions-informers ok
[+]poststarthook/start-apiextensions-controllers ok
[+]poststarthook/crd-informer-synced ok
[+]poststarthook/bootstrap-controller ok
[+]poststarthook/rbac/bootstrap-roles ok
[+]poststarthook/scheduling/bootstrap-system-priority-classes ok
[+]poststarthook/priority-and-fairness-config-producer ok
[+]poststarthook/start-cluster-authentication-info-controller ok
[+]poststarthook/aggregator-reload-proxy-client-cert ok
[+]poststarthook/start-kube-aggregator-informers ok
[+]poststarthook/apiservice-registration-controller ok
[+]poststarthook/apiservice-status-available-controller ok
[+]poststarthook/kube-apiserver-autoregistration ok
[+]autoregister-completion ok
[+]poststarthook/apiservice-openapi-controller ok
[+]poststarthook/apiservice-openapiv3-controller ok
healthz check passed

```
### Create TLS pairs

```
cd configmap/kong/
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ingress-admin.key -out ingress-admin.crt -subj "/CN=admin.kong.192.168.1.100.nip.io/O=admin.kong.admin.kong.192.168.1.100.nip.io"
```

### Pre-requisites
```
kubectl create namespace kong
kubectl create secret generic kong-superuser-password -n kong --from-literal=password=changeit
kubectl create secret tls ingress-admin-tls-secret --key ./configmap/kong/ingress-admin.key --cert ./configmap/kong/ingress-admin.crt -n kong
```

### Install Kong Ingress Controller and Konga UI
```
helm repo add kong https://charts.konghq.com
helm install my-kong kong/kong -n kong --values ./charts/kong/minimal.yml
helm install konga ./charts/konga -n kong --values ./charts/konga/values.yml
---> NO execute! (db migration needed on init kong): kubectl delete jobs -n kong --all

### Note:  We can install Kong Api gateway this way without step to setup TLS certs: 
$ helm install api-gateway -n kong kong/kong \
  --set ingressController.installCRDs=false \
  --set admin.enabled=true \
  --set admin.type=ClusterIP \
  --set admin.http.enabled=true \
  --set admin.tls.enabled=false \
  --set env.database=postgres \
  --set postgresql.enabled=true

```
Example Output:
```
$ helm install my-kong kong/kong -n kong --values ./charts/kong/minimal.yml
coalesce.go:223: warning: destination for kong.proxy.stream is a table. Ignoring non-table value ([])
coalesce.go:223: warning: destination for kong.proxy.stream is a table. Ignoring non-table value ([])
NAME: my-kong
LAST DEPLOYED: Wed Jul  5 21:47:25 2023
NAMESPACE: kong
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
To connect to Kong, please execute the following commands:
HOST=$(kubectl get nodes --namespace kong -o jsonpath='{.items[0].status.addresses[0].address}')
PORT=$(kubectl get svc --namespace kong my-kong-kong-proxy -o jsonpath='{.spec.ports[0].nodePort}')
export PROXY_IP=${HOST}:${PORT}
curl $PROXY_IP

Once installed, please follow along the getting started guide to start using
Kong: https://docs.konghq.com/kubernetes-ingress-controller/latest/guides/getting-started/

$ helm install konga ./charts/konga -n kong --values ./charts/konga/values.yml
NAME: konga
LAST DEPLOYED: Wed Jul  5 21:47:38 2023
NAMESPACE: kong
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace kong -l "app.kubernetes.io/name=konga,app.kubernetes.io/instance=konga" -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl port-forward $POD_NAME 8080:80
```

### Kong Ingress fix
```
$ kubectl get ing -n kong
NAME                 CLASS    HOSTS                             ADDRESS   PORTS     AGE
my-kong-kong-admin   <none>   admin.kong.192.168.1.100.nip.io             80, 443   35s
```
Note: ADDRESS is empty!!!

Fix ---> Ref: https://kind.sigs.k8s.io/docs/user/ingress/

```
kubectl apply -f https://raw.githubusercontent.com/Kong/kubernetes-ingress-controller/master/deploy/single/all-in-one-dbless.yaml
kubectl patch deployment -n kong proxy-kong -p '{"spec":{"replicas":1,"template":{"spec":{"containers":[{"name":"proxy","ports":[{"containerPort":8e3,"hostPort":80,"name":"proxy","protocol":"TCP"},{"containerPort":8443,"hostPort":443,"name":"proxy-ssl","protocol":"TCP"}]}],"nodeSelector":{"ingress-ready":"true"},"tolerations":[{"key":"node-role.kubernetes.io/control-plane","operator":"Equal","effect":"NoSchedule"},{"key":"node-role.kubernetes.io/master","operator":"Equal","effect":"NoSchedule"}]}}}}'
kubectl patch service -n kong kong-proxy -p '{"spec":{"type":"NodePort"}}'
kubectl patch ingress my-kong-kong-admin -p '{"spec":{"ingressClassName":"kong"}}' -n kong

### Check Ingress:
$ kubectl get ing -n kong
NAME                 CLASS   HOSTS                             ADDRESS         PORTS     AGE
my-kong-kong-admin   kong    admin.kong.192.168.1.100.nip.io   10.96.112.209   80, 443   2m43s

```

### Check kong installation

```
kubectl get all -n kong
NAME                                     READY   STATUS      RESTARTS   AGE
pod/ingress-kong-7548b68cb8-l9xpf        1/1     Running     0          103s
pod/konga-58bb6bd9f7-hkpqf               1/1     Running     0          3m30s
pod/my-kong-kong-8db7f644d-k64bq         1/1     Running     0          3m30s
pod/my-kong-kong-init-migrations-wrr77   0/1     Completed   0          3m30s
pod/my-kong-postgresql-0                 1/1     Running     0          3m30s
pod/proxy-kong-8d85c874f-4cssx           1/1     Running     0          103s
pod/proxy-kong-8d85c874f-qcxm4           1/1     Running     0          103s

NAME                              TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/kong-admin                ClusterIP      None            <none>        8444/TCP                     105s
service/kong-proxy                NodePort       10.96.112.209   <none>        80:32633/TCP,443:30859/TCP   104s
service/kong-validation-webhook   ClusterIP      10.96.35.199    <none>        443/TCP                      104s
service/konga                     ClusterIP      10.96.47.167    <none>        1337/TCP                     3m30s
service/my-kong-kong-admin        NodePort       10.96.243.68    <none>        8444:30447/TCP               3m30s
service/my-kong-kong-proxy        LoadBalancer   10.96.226.163   <pending>     80:32460/TCP,443:30840/TCP   3m30s
service/my-kong-postgresql        ClusterIP      10.96.97.49     <none>        5432/TCP                     3m30s
service/my-kong-postgresql-hl     ClusterIP      None            <none>        5432/TCP                     3m30s

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-kong   1/1     1            1           104s
deployment.apps/konga          1/1     1            1           3m30s
deployment.apps/my-kong-kong   1/1     1            1           3m30s
deployment.apps/proxy-kong     2/2     2            2           103s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/ingress-kong-7548b68cb8   1         1         1       104s
replicaset.apps/konga-58bb6bd9f7          1         1         1       3m30s
replicaset.apps/my-kong-kong-8db7f644d    1         1         1       3m30s
replicaset.apps/proxy-kong-8d85c874f      2         2         2       103s

NAME                                  READY   AGE
statefulset.apps/my-kong-postgresql   1/1     3m30s

NAME                                     COMPLETIONS   DURATION   AGE
job.batch/my-kong-kong-init-migrations   1/1           2m54s      3m30s


$ kubectl get ing -n kong
NAME                 CLASS   HOSTS                             ADDRESS         PORTS     AGE
my-kong-kong-admin   kong    admin.kong.192.168.1.100.nip.io   10.96.112.209   80, 443   3m56s

$ HOST=$(kubectl get nodes --namespace kong -o jsonpath='{.items[0].status.addresses[0].address}')
$ PORT=$(kubectl get svc --namespace kong my-kong-kong-proxy -o jsonpath='{.spec.ports[0].nodePort}')
$ export PROXY_IP=${HOST}:${PORT}
$ echo $PROXY_IP
172.20.0.3:32460

$ curl $PROXY_IP
{"message":"no Route matched with those values"}

$ kubectl port-forward service/my-kong-kong-admin 8080:8444 -n kong
Forwarding from 127.0.0.1:8080 -> 8444
Forwarding from [::1]:8080 -> 8444
Handling connection for 8080
Handling connection for 8080
E0705 22:01:12.880949  113008 portforward.go:391] error copying from local connection to remote stream: read tcp4 127.0.0.1:8080->127.0.0.1:40100: read: connection reset by peer
Handling connection for 8080

$ curl -k https://localhost:8080
{"tagline":"Welcome to kong","plugins":{"available_on_server":{"cors":true,"oauth2":true,"tcp-log":true,"udp-log":true,"file-log":true,"http-log":true,"key-auth":true,"hmac-auth":true,"basic-auth":true,"ip-restriction":true,"request-transformer":true,"response-
ecdsa.key","nginx_pid":"/kong_prefix/pids/nginx.pid","nginx_err_logs":"/kong_prefix/logs/error.log","portal_reset_success_email":true,"admin_acc_logs":"/kong_prefix/logs/admin_access.log","smtp_host":"localhost","vitals_statsd_udp_packet_size":1024,"dns_no_sync":false},"timers":
...
...
{"pending":13,"running":0},"node_id":"6c119dbe-2eed-4be8-91e1-dc4ccd5738a6","version":"2.7.0.0-enterprise-edition","lua_version":"LuaJIT 2.1.0-beta3"}


$ export KONG_PROXY_LB=$(kubectl get svc/my-kong-kong-proxy -n kong -o=jsonpath='{.spec.clusterIP}')
$ echo $KONG_PROXY_LB
10.96.226.163
$ export KONG_ADMIN_POD=$(kubectl get pod --selector=app=my-kong-kong -n kong -o=jsonpath='{.items[0].status.podIP}')
$ echo $KONG_ADMIN_POD
10.244.1.2
$ export KONG_GATEWAY_DOMAIN=apigw.kong.192.168.1.100.nip.io
$ curl https://$KONG_ADMIN_POD:8444
^C
$ kubectl edit ing/my-kong-kong-admin -n kong
$ KONG_ADMIN_LISTEN=localhost:8444
```

### KONGA 
```
$ kubectl port-forward service/konga 8080:1337 -n kong
Forwarding from 127.0.0.1:8080 -> 1337
Forwarding from [::1]:8080 -> 1337
Handling connection for 8080
Handling connection for 8080
Handling connection for 8080
Handling connection for 8080
Handling connection for 8080
```

Browser -> Konga UI : http://127.0.0.1:8080 -> Create admin user

Screenshot:

<img src="screenshots/Konga-UI-create-admin.png?raw=true" width="1000">


#### Initialize Kong Connection for Konga UI

```
$ echo $KONG_ADMIN_POD
10.244.1.2
```

Add New Connection ---> KONG ADMIN URL = https://10.244.1.2:8444

Screenshots:


<img src="screenshots/Konga-UI-Connections.png?raw=true" width="1000">

<img src="screenshots/Konga-UI-DASHBOARD.png?raw=true" width="1000">

#### Apply echo pod/service for demonstration
`kubectl apply -f ./manifests/echo.yml`

#### Get IPs for Kong Proxy LB and Kong Admin
`export KONG_PROXY_LB=$(kubectl get svc/my-kong-kong-proxy -n kong -o=jsonpath='{.spec.clusterIP}')` \
`export KONG_ADMIN_POD=$(kubectl get pod --selector=app=my-kong-kong -n kong -o=jsonpath='{.items[0].status.podIP}')`

Example Output
```
$ export KONG_PROXY_LB=$(kubectl get svc/my-kong-kong-proxy -n kong -o=jsonpath='{.spec.clusterIP}')
$ echo $KONG_PROXY_LB
10.96.226.163
$ export KONG_ADMIN_POD=$(kubectl get pod --selector=app=my-kong-kong -n kong -o=jsonpath='{.items[0].status.podIP}')
$ echo $KONG_ADMIN_POD
10.244.1.2

```

#### Set Kong Gateway Domain
`export KONG_GATEWAY_DOMAIN=apigw.kong.192.168.1.100.nip.io`

#### Add service/route
__Service__
Name: echo-foo \
Protocol: http \
Host: echo.default \
Port: 80 \
Path: /foo \

__Route__ (Note: ENTER when setup path and hosts)
Name: echo-foo-route \
Hosts: `$KONG_GATEWAY_DOMAIN` \
Path: /echo/foo


Screenshots:

<img src="screenshots/Konga-UI-service.png?raw=true" width="1000">

<img src="screenshots/Kong-UI-route.png?raw=true" width="1000">

<img src="screenshots/Konga-UI-add-route.png?raw=true" width="1000">


#### Add DNS for Kong Proxy LB
`kubectl edit cm/coredns -n kube-system` \

NodeHosts: \
... \
... \
`$KONG_PROXY_LB $KONG_GATEWAY_DOMAIN`

#### HTTP and HTTPS Connectivity Testing
`kubectl exec -it my-kong-postgresql-0 -n kong -- curl http://apigw.kong.192.168.1.100.nip.io/echo/foo` \
`kubectl exec -it my-kong-postgresql-0 -n kong -- curl -k https://apigw.kong.192.168.1.100.nip.io/echo/foo`

### Some features of Kong Ingress Controller and Kong Gateway:

- Proxy service by Kong
- Protect API by API Key
- Limit API traffic
- Restrict IP Addresses of API consumer
- Setup Backend Services Health Check and Circuit-Breaker

### Clean env:
```
kind delete cluster --name=kong-cluster
```
