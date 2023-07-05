## Kong API Gateway on KinD
This guide introduces anyone who is interested in doing local hands-on experience with Kong Gateway on KinD (or k3d) cluster

### Dependencies

- [Docker](https://docs.docker.com/engine/install/ubuntu/)
- [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/docs/intro/install/)

### Create k8s cluster (KinD)
```
cd kind/
kind create cluster --name=kong-cluster --config=config.yml
```
Note: Example using k3d to create k3d cluster without traefik as default ingress class `k3d cluster create sandman -p 8888:80@loadbalancer -v /etc/machine-id:/etc/machine-id:ro -v /var/log/journal:/var/log/journal:ro -v /var/run/docker.sock:/var/run/docker.sock --k3s-arg '--disable=traefik@server:0' --agents 0`

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
---> NO execute!: kubectl delete jobs -n kong --all
```

Note: Kong proxy (Fix LoadBalancer and try with k3d)
```
charts/kong/minimal.yml

proxy:
  enabled: true
  type: NodePort
  annotations: {}

Change to:

proxy:
  enabled: true
  type: LoadBalancer
  annotations: {}


$ kubectl get all -n kong
NAME                                     READY   STATUS      RESTARTS   AGE
pod/konga-594884cb7c-5hrfx               1/1     Running     0          9m53s
pod/my-kong-postgresql-0                 1/1     Running     0          10m
pod/my-kong-kong-init-migrations-wd8fh   0/1     Completed   0          10m
pod/my-kong-kong-5bbf57dbfd-7n8qz        1/1     Running     0          10m

NAME                            TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                      AGE
service/my-kong-postgresql-hl   ClusterIP      None            <none>          5432/TCP                     10m
service/my-kong-postgresql      ClusterIP      10.43.37.135    <none>          5432/TCP                     10m
service/my-kong-kong-admin      NodePort       10.43.250.157   <none>          8444:32371/TCP               10m
service/konga                   ClusterIP      10.43.94.172    <none>          1337/TCP                     9m53s
service/my-kong-kong-proxy      LoadBalancer   10.43.43.239    192.168.192.2   80:32500/TCP,443:30134/TCP   10m

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/konga          1/1     1            1           9m53s
deployment.apps/my-kong-kong   1/1     1            1           10m

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/konga-594884cb7c          1         1         1       9m53s
replicaset.apps/my-kong-kong-5bbf57dbfd   1         1         1       10m

NAME                                  READY   AGE
statefulset.apps/my-kong-postgresql   1/1     10m

NAME                                     COMPLETIONS   DURATION   AGE
job.batch/my-kong-kong-init-migrations   1/1           4m14s      10m
$ kubectl get ing -n kong
NAME                 CLASS    HOSTS                             ADDRESS   PORTS     AGE
my-kong-kong-admin   <none>   admin.kong.192.168.1.100.nip.io             80, 443   10m

Note: ADDRESS is empty with KinD & k3d! (TODO: Fix) 
```

Example Output:
```
charts/kong/minimal.yml

proxy:
  enabled: true
  type: LoadBalancer
  annotations: {}

Change to:

proxy:
  enabled: true
  type: NodePort
  annotations: {}


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
### Check kong installation

```
$ kubectl get all -n kong
NAME                                     READY   STATUS      RESTARTS   AGE
pod/konga-58bb6bd9f7-xlzrs               1/1     Running     0          7m51s
pod/my-kong-kong-8db7f644d-svkhv         1/1     Running     0          8m3s
pod/my-kong-kong-init-migrations-6hckw   0/1     Completed   0          8m3s
pod/my-kong-postgresql-0                 1/1     Running     0          8m3s

NAME                            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
service/konga                   ClusterIP   10.96.136.94   <none>        1337/TCP                     7m51s
service/my-kong-kong-admin      NodePort    10.96.11.253   <none>        8444:31342/TCP               8m3s
service/my-kong-kong-proxy      NodePort    10.96.166.5    <none>        80:32417/TCP,443:32598/TCP   8m3s
service/my-kong-postgresql      ClusterIP   10.96.6.84     <none>        5432/TCP                     8m3s
service/my-kong-postgresql-hl   ClusterIP   None           <none>        5432/TCP                     8m3s

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/konga          1/1     1            1           7m51s
deployment.apps/my-kong-kong   1/1     1            1           8m3s

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/konga-58bb6bd9f7         1         1         1       7m51s
replicaset.apps/my-kong-kong-8db7f644d   1         1         1       8m3s

NAME                                  READY   AGE
statefulset.apps/my-kong-postgresql   1/1     8m3s

NAME                                     COMPLETIONS   DURATION   AGE
job.batch/my-kong-kong-init-migrations   1/1           110s       8m3s

$ kubectl get ing -n kong
NAME                 CLASS    HOSTS                             ADDRESS   PORTS     AGE
my-kong-kong-admin   <none>   admin.kong.192.168.1.100.nip.io             80, 443   8m15s

$ HOST=$(kubectl get nodes --namespace kong -o jsonpath='{.items[0].status.addresses[0].address}')
$ echo $HOST
172.20.0.3
$ PORT=$(kubectl get svc --namespace kong my-kong-kong-proxy -o jsonpath='{.spec.ports[0].nodePort}')
$ echo $PORT
32417
$ export PROXY_IP=${HOST}:${PORT}
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
10.96.166.5
$ export KONG_ADMIN_POD=$(kubectl get pod --selector=app=my-kong-kong -n kong -o=jsonpath='{.items[0].status.podIP}')
$ echo $KONG_ADMIN_POD
10.244.2.2
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
10.244.2.2
```

Add New Connection ---> KONG ADMIN URL = https://10.244.2.2:8444

Screenshots:

<img src="screenshots/Konga-UI-all-CONNECTIONS.png?raw=true" width="1000">

<img src="screenshots/Konga-UI-CONNECTIONS.png?raw=true" width="1000">

#### Apply echo pod/service for demonstration
`kubectl apply -f ./manifests/echo.yml`

#### Get IPs for Kong Proxy LB and Kong Admin
`export KONG_PROXY_LB=$(kubectl get svc/my-kong-kong-proxy -n kong -o=jsonpath='{.spec.clusterIP}')` \
`export KONG_ADMIN_POD=$(kubectl get pod --selector=app=my-kong-kong -n kong -o=jsonpath='{.items[0].status.podIP}')`

Example Output
```
$ export KONG_PROXY_LB=$(kubectl get svc/my-kong-kong-proxy -n kong -o=jsonpath='{.spec.clusterIP}')
$ echo $KONG_PROXY_LB
10.96.166.5
$ export KONG_ADMIN_POD=$(kubectl get pod --selector=app=my-kong-kong -n kong -o=jsonpath='{.items[0].status.podIP}')
$ echo $KONG_ADMIN_POD
10.244.2.2

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

__Route__
Name: echo-foo-route \
Hosts: `$KONG_GATEWAY_DOMAIN` \
Path: /echo/foo


Screenshots:

<img src="screenshots/Konga-UI-service.png?raw=true" width="1000">

<img src="screenshots/Kong-UI-route.png?raw=true" width="1000">



#### Add DNS for Kong Proxy LB
`kubectl edit cm/coredns -n kube-system` \

NodeHosts: \
... \
... \
`$KONG_PROXY_LB $KONG_GATEWAY_DOMAIN`

#### HTTP and HTTPS Connectivity Testing
`kubectl exec -it my-kong-postgresql-0 -n kong -- curl http://apigw.kong.192.168.1.100.nip.io/echo/foo` \
`kubectl exec -it my-kong-postgresql-0 -n kong -- curl -k https://apigw.kong.192.168.1.100.nip.io/echo/foo`
