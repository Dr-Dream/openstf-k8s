# Dynamic provider proxy for OpenSTF deployment in k8s
This is simple proxy based on NGINX that maps providers WebSocket urls to concrete 'external' provider.

This achived throug fact that provider public ip is part of the screen-ws-url-pattern. In normal situation your provider launc command looks like this:
```
stf provider --name "<PROVIDER_NAME>" \
	     --connect-sub "tcp://<DEV_TRIPROXY_IP>:<DEV_TRIPROXY_SUB_PORT>" \
             --connect-push "tcp://<DEV_TRIPROXY_IP>:<DEV_TRIPROXY_PUB_PORT>" \
             --storage-url "<STF_BASE_URL>" \
             --screen-ws-url-pattern "wss://<OPENSTF_BASE>/d/%i/<%= serial %>/<%= publicPort %>/" \ 
             --public-ip <PROVIDER_IP> \
             --min-port=15000 \
             --max-port=25000 \
             --heartbeat-interval 10000
```
So we replacing it with following:
```
stf provider --name "<PROVIDER_NAME>" \
             --connect-sub "tcp://<DEV_TRIPROXY_IP>:<DEV_TRIPROXY_SUB_PORT>" \
             --connect-push "tcp://<DEV_TRIPROXY_IP>:<DEV_TRIPROXY_PUB_PORT>" \
             --storage-url "<STF_BASE_URL>" \
             --screen-ws-url-pattern "wss://<OPENSTF_BASE>/d/dynamic/<%= serial %>/<%= publicPort %>/<%= publicIp %>/" \
             --public-ip <PROVIDER_IP>  \
             --min-port=15000 \
             --max-port=25000 \
             --heartbeat-interval 10000
```
* We replaced %i to 'dynamic' to route urls to this proxy in ingress
* And added <%= publicIp %>/ to the end of screen-ws-url-pattern so proxy could find which host exactly it should connect.
Note: PROVIDER_IP - must be accessable for k8s nodes which runs STF.

## Setting-Up

### Building docker image
Nothing special.
* In 'docker' directory
* Fix nginx.conf for domain name.
* Build docker image for openstf-provider-proxy
```
docker build -t openstf-provider-proxy .
docker tag openstf-provider-proxy <REPOSITORY_IP:REPOSITORY_PORT>/openstf/openstf-provider-proxy:latest
docker push <REPOSITORY_IP:REPOSITORY_PORT>/openstf/openstf-provider-proxy:latest
```
* Install in k8s 
```
kubectl -n openstf apply -f .
```
### Private repositories
Please check imagePullSecret in [openstf-provider-proxy.yaml](openstf-provider-proxy.yaml) for Replication controller
And create required sercret
```
kubectl -n openstf create secret docker-registry nexus3-docker-secret --docker-server=<REGISTRY_SERVER:PORT> --docker-username="<USERNAME>" --docker-password="<PASSWORD>" --docker-email="<EMAIL>"
```
### Public repositories
Just remove Image Pull Secret from [openstf-provider-proxy.yaml](openstf-provider-proxy.yaml)

Please, check [openstf-ingress](../openstf-ingress) . Ingress will map 'd/dynamic/' contained urls to that service.
