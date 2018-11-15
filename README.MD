# OpenSTF Deployment on K8S without Helm

This is simple manifests to run [STF](https://github.com/openstf/stf) on [k8s](https://kubernetes.io/) without helm.
Most of them are copied with some modifications from [stf-k8s](https://github.com/dimasaryo/stf-k8s) by [Dimas Aryo Prakoso](https://github.com/dimasaryo)
One addition is that [openstf provider proxy](k8s/nginx-provider-proxy/openstf-provider-proxy.yaml), which is dynamicly routes screen WebSockets to provider, that is outside of k8s.
This is something that was partially reworked by me to get clearance on what is [STF](https://github.com/openstf/stf), how it works, and have some fun with K8S.

!!! It is NOT secured enough !!!

!!! It is NOT robust. (simple RethinkDB, no liveness/readiness probes and etc.) !!!


But it was good quick start, and i hope it will help someone.

## In this manifests
* Domain dev.example.ru used as accessable domain so it is supposed that STF will be deployed on openstf.dev.example.ru domain
* Secrets:
   * "RETHINKDB_DB_SECRET" - as RethinkDB authkey
   * "OPENSTF_SECRET" - as STF web token secret.
 
## Setting Up

### What you need to do first.
1) Please check [rethinkdb](k8s/rethinkdb).
   This is basic single node rethinkdb deployment. So it really not recomended to use it in production, due low security. In this deployment rethink db is used as an example for quick start. Also it contains ingress that exposes admin intrface to public. So be aware on that.

   Most important you need to know related to OpenSTF is that 'rethinkdb-secret' in ([openstf-config](k8s/openstf-config)) is created and it's reused by all services as AUTHKEY to access rethinkdb. Please note that same AUTHKEY must be applyed to 'admin' user by running following query (ex. throug admin interface).
```
r.db('rethinkdb').table('users').get('admin').update({password:'RETHINKDB_DB_SECRET'})
```
   Otherwise services will be not able to connect rethinkdb.
   Also this secret used by migrate job.

2) Build and deploy [openstf provider proxy](k8s/nginx-provider-proxy/docker/) image to your registry. You could find description what it is in [this README.MD](k8s/nginx-provider-proxy/README.MD).

3) In [openstf-config](openstf-config) 'openstf-app-secret' is created which is used as security token for all web services. So it's populated as environment variable to all services like 'api', 'app', 'storage', 'websocket' and etc. 

4) Check [openstf-config.yaml](k8s/openstf-config/openstf-config.yaml) and correct domain names and rethinkdb url.

5) **[openstf-storage-temp](k8s/openstf-storage-temp)** and **[rethinkdb](rethinkdb)** requires persistent volume claims, so look into subfolders to create required claims. So following claims 'openstf-storage-temp-data-claim' and 'rethinkdb-data-claim' are required. Refer to [k8s documentation](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) on persistent volumes.

6) Be aware that [openstf-auth](k8s/openstf-auth) is **MOCK AUTH**, so it is absolutelly **NOT SECURED**. So it is not recommended for production. Please refer to [original documentation](https://github.com/openstf/stf/blob/master/doc/DEPLOYMENT.md#stf-authservice) to setup another authentication environment.

### Applying on k8s

In basic setup you just need to apply everything in 'k8s' subfolders
* [openstf-config](k8s/openstf-config)
* [openstf-config](k8s/openstf-config)
* [openstf-api](k8s/openstf-api)
* [openstf-app](k8s/openstf-app)
* [openstf-auth](k8s/openstf-auth)
* [openstf-ingress](k8s/openstf-ingress)
* [openstf-migrate](k8s/openstf-migrate)
* [openstf-processor](k8s/openstf-processor)
* [openstf-reaper](k8s/openstf-reaper)
* [openstf-storage-plugin-apk](k8s/openstf-storage-plugin-apk)
* [openstf-storage-plugin-image](k8s/openstf-storage-plugin-image)
* [openstf-storage-temp](k8s/openstf-storage-temp)
* [openstf-triproxy-app](k8s/openstf-triproxy-app)
* [openstf-triproxy-dev](k8s/openstf-triproxy-dev)
* [openstf-websocket](k8s/openstf-websocket)
* [rethinkdb](k8s/rethinkdb)

**Except docker-provider**
[docker-provider](docker-provider) is a docker-compose to run on machine outside of your k8s installation.

If you already have rethinkdb cluster, than skip applying [rethinkdb](k8s/rethinkdb) manifests.

Basicly run

```
kubectl -n openstf apply -f .
```
in each sub directory
Or just run 
```
kubectl -n openstf apply -R -f k8s
```

Note: Be aware that openstf is namespace, that should exists.

### Post actions
As it was mentioned earlier, please, create Persistent Volume Claims 'openstf-storage-temp-data-claim' and 'rethinkdb-data-claim', if it was not done earlier. Refer to [k8s documentation](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) on persistent volumes.

### Running a provider.
It's simple.
1) Get Linux Machine, that you'll connect your devices.
2) Install [docker](https://docs.docker.com/install/) and [docker-compose](https://docs.docker.com/compose/install/).
3) Copy [docker-provider](docker-provider) to somewhere on that machine.
4) Fix [docker-compose.yaml](docker-provider/docker-compose.yaml). Change 'openstf-provider' service with your domains, ips and ports.
```
docker-compose up -d
```
5) Connect your devices
6) Enjoy.

## References
* OpenSTF/stf project - <https://github.com/openstf/stf>
   Original STF. Better to start from there. Many thanks to authors for their work.
* stf-k8s project -<https://github.com/dimasaryo/stf-k8s>
   Helps a lot for first deployment.
* Agoda Android Farm <https://github.com/agoda-com/android-farm>
   Helm based deployment. More sofisticated approach. Includes emulators, liveness probes etc. Will be next step.