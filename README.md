# openshift-istio-playground
Playground for testing out some istio features and configs

#### Vanilla SM2
`vanilla-sm2` folder contains the basic installation files for running istio / service-mesh-2 on Openshift namely: Subscriptions, Namespaces, Control Plane and Member Roll.

The service-mesh only controls two namespaces: `istio-system` and `vanilla-playground`.

It deploys a `hello-openshift` application alongside its own service. However, to access the application the service-mesh needs to be configured, therefore, the application has an annotation to add the sidecar. The traffic should go through the route istio-ingressgateway, which contains the routing rules to route all the requests.

##### Deployment
```bash
oc apply -f vanilla-sm2/namespace.yaml
oc apply -f vanilla-sm2/subscription.yaml
oc apply -f vanilla-sm2/istio.yaml
oc apply -f vanilla-sm2/app.yaml
```

##### Application
```bash
oc project istio-system
oc get route/istio-ingressgateway
# test the istio-ingressgateway url, it should hit the deployed application
curl istio-ingressgateway-istio-system.apps.cluster-rwkvp.rwkvp.sandbox2991.opentlc.com
Hello OpenShift!
```

##### References

1. [ServiceMesh2 Documention for Openshift 4.18](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/service_mesh/service-mesh-2-x#ossm-member-roll-create-cli_ossm-create-mesh)


#### Testing out the warmupDurationSecs in the DestinationRule

#### Allow pods to make requests to external hosts (external gateways)
> The `spec.proxy.networking.trafficControl.outbound.policy configuration value in
the ServiceMeshControlPlane resource controls this behavior. The default value for this entry is ALLOW_ANY. This value instructs Istio to allow all egress traffic regardless of the destination. If this configuration holds the value REGISTRY_ONLY, the gateway only forwards requests to services explicitly registered.

1. Define outboundTrafficPolicy.mode = ALLOW_ANY / REGISTRY_ONLY
```bash
$ oc get ServiceMEshControlPlane -n istio-system
NAME    READY   STATUS            PROFILES      VERSION   AGE
basic   8/8     ComponentsReady   ["default"]   2.6.7     45m
$ oc edit ServiceMEshControlPlane/basic -n istio-system
...
apiVersion: maistra.io/v2
kind: ServiceMeshControlPlane
spec:
  proxy:
    networking:
      trafficControl:
        outbound:
          policy: REGISTRY_ONLY
...
$ oc exec "$SOURCE_POD" -c sleep -- curl -sSI https://www.google.com | grep  "HTTP/"; kubectl exec "$SOURCE_POD" -c sleep -- curl -sI https://edition.cnn.com | grep "HTTP/"
HTTP/2 200
```
2. Create a ServiceEntry resource to register an external service to the mesh.
```bash
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: google
spec:
  hosts:
  - httpbin.org
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
  location: MESH_EXTERNAL
```
3. Create a VirtualService	to define routing rules for traffic.
```bash
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin-ext
spec:
  hosts:
    - httpbin.org
  http:
  - timeout: 3s
    route:
      - destination:
          host: httpbin.org
        weight: 100
```
4. Test querying httpbin from the pod
```bash
$ oc exec "$SOURCE_POD" -c sleep -- curl -sSI https://www.google.com | grep  "HTTP/"; kubectl exec "$SOURCE_POD" -c sleep -- curl -sI https://edition.cnn.com | grep "HTTP/"
```

#### Disable readiness in the istio monitor
```bash
# it restarts the container (will be temporarly disconnect form the mesh)
$ oc -n vanilla-playground exec -ti deploy/hello-openshift -c istio-proxy -- curl -X POST localhost:15000/quitquitquit

# it removes the container from the mesh (not suggested)
$ oc -n vanilla-playground exec -ti deploy/hello-openshift -c istio-proxy -- curl -X POST localhost:15000/drain_listeners

# other option, it forces the /ready to be false
$ oc -n vanilla-playground exec -ti deploy/hello-openshift -c istio-proxy -- curl -X POST localhost:15000/ready           
LIVE
$ oc -n vanilla-playground exec -ti deploy/hello-openshift -c istio-proxy -- curl -X POST localhost:15000/healthcheck/fail
OK
$ oc -n vanilla-playground exec -ti deploy/hello-openshift -c istio-proxy -- curl -X POST localhost:15000/ready           
DRAINING
$ oc -n vanilla-playground exec -ti deploy/hello-openshift -c istio-proxy -- curl -X POST localhost:15000/healthcheck/ok  
OK
```
#### Check the gateway configuration
After applying the Virtual Services, they are merged into a configuration file in the ingressgateway pod.

```bash
$ oc project istio-system
$ oc get pods
NAME                                    READY   STATUS    RESTARTS   AGE
grafana-59c8485d-jcdq6                  2/2     Running   0          7m57s
istio-egressgateway-5fc86c577d-6mdnm    1/1     Running   0          7m57s
istio-ingressgateway-74ddf8dd9f-x8nls   1/1     Running   0          7m57s
istiod-basic-67469bcdb5-l5vzf           1/1     Running   0          8m38s
kiali-66d96cf774-gk5dc                  1/1     Running   0          6m11s
prometheus-748c6fdff8-qwbb8             3/3     Running   0          8m28s
$ oc rsh istio-ingressgateway-74ddf8dd9f-x8nls curl localhost:15000/config_dump
...
"filter_chains": [
        {
         "filters": [
          {
           "name": "envoy.filters.network.http_connection_manager",
           "typed_config": {
            "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager",
            "stat_prefix": "stats",
            "route_config": {
             "virtual_hosts": [
              {
               "name": "backend",
               "domains": [
                "*"
               ],
               "routes": [
                {
                 "match": {
                  "prefix": "/stats/prometheus"
                 },
                 "route": {
                  "cluster": "prometheus_stats"
                 }
                }
               ]
              }
             ]
            },
            "http_filters": [
             {
              "name": "envoy.filters.http.router",
              "typed_config": {
               "@type": "type.googleapis.com/envoy.extensions.filters.http.router.v3.Router"
              }
             }
...
```

#### Check sidecar configuration

```bash 
oc exec hello-openshift-5b7bcb9b6d-t8ctp -c istio-proxy curl localhost:15000/config_dump
...
{
     "version_info": "2025-06-02T13:56:24Z/15",
     "cluster": {
      "@type": "type.googleapis.com/envoy.config.cluster.v3.Cluster",
      "name": "outbound|15010||istiod-basic.istio-system.svc.cluster.local",
      "type": "EDS",
      "eds_cluster_config": {
       "eds_config": {
        "ads": {},
        "initial_fetch_timeout": "0s",
        "resource_api_version": "V3"
       },
       "service_name": "outbound|15010||istiod-basic.istio-system.svc.cluster.local"
      },
      "connect_timeout": "10s",
      "lb_policy": "LEAST_REQUEST",
      "circuit_breakers": {
       "thresholds": [
        {
         "max_connections": 4294967295,
         "max_pending_requests": 4294967295,
         "max_requests": 4294967295,
         "max_retries": 4294967295,
         "track_remaining": true
        }
       ]
      },
      "metadata": {
       "filter_metadata": {
        "istio": {
         "services": [
          {
           "name": "istiod-basic",
           "host": "istiod-basic.istio-system.svc.cluster.local",
           "namespace": "istio-system"
          }
         ],
         "config": "/apis/networking.istio.io/v1alpha3/namespaces/istio-system/destination-rule/istiod-basic"
        }
       }
      }
...
```

##### References
1. [Set port to zero - issues/github](https://github.com/istio/istio/issues/9504#issuecomment-439432130)