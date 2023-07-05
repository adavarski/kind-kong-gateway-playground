## Kong Gateway on k3d
This guide introduces anyone who is interested in doing local hands-on experience with Kong Gateway on k3d cluster

#### Create k3d cluster without traefik as default ingress class
`k3d cluster create k3s-local --k3s-arg '--no-deploy=traefik@server:*' --k3s-arg '--write-kubeconfig-mode=644@server:*' --servers 3`
```
cd kind/
kind create cluster --name=kong-cluster --config=config.yml
```
#### Create TLS pairs

```
cd configmap/kong/
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ingress-admin.key -out ingress-admin.crt -subj "/CN=admin.kong.192.168.1.100.nip.io/O=admin.kong.admin.kong.192.168.1.100.nip.io"
```

#### Pre-requisites
`kubectl create namespace kong` \
`kubectl create secret generic kong-superuser-password -n kong --from-literal=password=changeit` \ 
`kubectl create secret tls ingress-admin-tls-secret --key ./configmap/kong/ingress-admin.key --cert ./configmap/kong/ingress-admin.crt -n kong`

#### Install Kong Ingress Controller and Konga UI
`helm install my-kong kong/kong -n kong --values ./charts/kong/minimal.yml` \
`helm install konga ./charts/konga -n kong --values ./charts/konga/values.yml` \
`kubectl delete jobs -n kong --all`

#### Apply echo pod/service for demonstration
`kubectl apply -f ./manifests/echo.yml`

#### Get IPs for Kong Proxy LB and Kong Admin
`export KONG_PROXY_LB=$(kubectl get svc/my-kong-kong-proxy -n kong -o=jsonpath='{.spec.clusterIP}')` \
`export KONG_ADMIN_POD=$(kubectl get pod --selector=app=my-kong-kong -n kong -o=jsonpath='{.items[0].status.podIP}')`

#### Set Kong Gateway Domain
`export KONG_GATEWAY_DOMAIN=apigw.kong.deejiw.com`

#### Initialize Kong Connection for Konga UI
`https://$KONG_ADMIN_POD:8444`

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

#### Add DNS for Kong Proxy LB
`kubectl edit cm/coredns -n kube-system` \

NodeHosts: \
... \
... \
`$KONG_PROXY_LB $KONG_GATEWAY_DOMAIN`

#### HTTP and HTTPS Connectivity Testing
`kubectl exec -it my-kong-postgresql-0 -n kong -- curl http://apigw.kong.deejiw.com/echo/foo` \
`kubectl exec -it my-kong-postgresql-0 -n kong -- curl -k https://apigw.kong.deejiw.com/echo/foo`
