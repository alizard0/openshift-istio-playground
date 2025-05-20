# openshift-istio-playground
Playground for testing out some istio features and configs

#### Vanilla SM2
`vanilla-sm2` folder contains the basic installation files for running istio / service-mesh-2 on Openshift namely: Subscriptions, Namespaces, Control Plane and Member Roll.

The service-mesh only controls two namespaces: `istio-system` and `vanilla-playground`.

It deploys a `hello-openshift` application alongside its own service. However, to access the application the service-mesh needs to be configured, therefore, the application has an annotation to add the sidecar. As the ingress-gateway was configured with ClusterIP, it is required to use an Openshift Route to route traffic to the ingress-gateway (adding one `hop` to the networking flow).

This `hop` is configured by `istio-ingressgateway-route` which expects traffic on `http2`, redirects to the service `istio-ingressgateway` which will sends the requests to external-traffic-gw, then routed through external-traffic-vs and finally hits the pod.

##### Deployment
```bash
oc apply -f vanilla-sm2/namespace.yaml
oc apply -f vanilla-sm2/subscription.yaml
oc apply -f vanilla-sm2/istio.yaml
oc apply -f vanilla-sm2/app.yaml
```

##### References

1. [ServiceMesh2 Documention for Openshift 4.18](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/service_mesh/service-mesh-2-x#ossm-member-roll-create-cli_ossm-create-mesh)


#### Testing out the warmupDurationSecs in the DestinationRule

#### Allow pods to make requests to external hosts

#### Disable readiness in the istio monitor

##### References

1. [Set port to zero - issues/github](https://github.com/istio/istio/issues/9504#issuecomment-439432130)