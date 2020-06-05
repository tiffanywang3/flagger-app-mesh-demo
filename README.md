# Flagger and App Mesh Demo

In today's demo, we will be reviewing some how Flagger uses App Mesh to automatically, 
incrementally introduce traffic to an upgraded version of an application before promoting it 
to serve 100% of live traffic. We will also demonstrate what happens in a "failed" deployment
scenario, whereupon failed metrics, the attempt to upgrade the application halts and rolls back.

Tools used for this demo include:

* `eksctl` to create the cluster, `eksctl enable repo` to install GitOps Operator tools (FluxCD, Helm Operator),
`eksctl enable profile appmesh` to install:
        * Kubernetes custom resources: mesh, virtual nodes and virtual services
        * CRD controller: keeps the custom resources in sync with the App Mesh control plane
        * Admission controller: injects the Envoy sidecar and assigns pods to App Mesh virtual nodes
        * Telemetry service: Prometheus instance that collects and stores Envoy’s metrics
        * Progressive delivery operator: Flagger instance that automates canary releases on top of App Mesh
* `app mesh` as the service mesh (comprised of the App Mesh controller, CRDs, Grafana, 
sidecar injector, Prometheus)
* GitOps to ensure Git is the source of truth for declarative infrastructure and applications
* `fluxctl` to allow Flux to automatically apply and deploy contents of your Git repository
* `flagger` to implement Canary deployments and progressive delivery, automating the release process 
for applications running on Kubernetes

## Automated Canary Promotion

Today's demo uses a sample microservice `podinfo` to mimic an application for us to Canary. Version 3.1.0 is currently 
deployed, and as a mechanism to demonstrate a visual change, we will be editing the image displayed on the page as we edit 
the `podinfo` image tag. 

You can get the ingress public address by running:

```sh
export URL="http://$(kubectl -n demo get svc/appmesh-gateway -ojson | jq -r ".status.loadBalancer.ingress[].hostname")"
echo $URL
```

To demonstrate an automated Canary promotion, we will be making a very small commit that comprises of updating the image tag
to `3.1.1` as well as the `env` variable `PODINFO_UI_LOGO` to instead display `https://eks.handson.flagger.dev/cuddle_bunny.gif`.

A quick note here, this step of updating the image tag will typically be handled by Flux. Flux is able to monitor the image 
repository where you push your application images to, and can make this commit on your behalf. You can specify whether or 
not you want Flux to automate the release of every update using regular expressions in a Flux annotation. We won't be demoing this 
but it is possible for delivery to be automated safely once a development team has successfully built their application's image.

To watch the Canary progress, we will monitor the canaries via:

```sh
kubectl -n demo get canaries --watch
```

and the Flagger logs via:

```sh
kubectl -n appmesh-system logs deployment/flagger -f | jq .msg
```

We expect this upgrade to successfully progress with incremented traffic weights as specified in `canary.yaml`. At 50%, we will 
see the promotion begin to meet the HPA minimum (2 replicas, specified in `hpa.yaml`).

## Automated Canary Rollback

To demonstrate a faulty release scenario, we will be injecting some `HTTP 500` errors during the promotion of the Canary
to trigger an automated rollback.

We will edit the image tag to `3.1.2`. 

To fabricate some failed requests, we'll first `exec` into the `flagger-loadtester` pod:

```sh
kubectl -n demo exec -it $(kubectl -n demo get pods -o name | grep -m1 flagger-loadtester | cut -d'/' -f 2) bash
```

We'll use `hey` to generate some HTTP 500 requests and cause some delays:

```sh
hey -z 1m -c 5 -q 5 http://podinfo-canary.demo:9898/status/500 && \
hey -z 1m -c 5 -q 5 http://podinfo-canary.demo:9898/delay/1
```

To watch the Canary progress, we will monitor the canaries via:

```sh
kubectl -n demo get canaries --watch
```

and the Flagger logs via:

```sh
kubectl -n appmesh-system logs deployment/flagger -f | jq .msg
```

In this example, we'll expect to see the Canary progress until the errors take place. Once the threshold for errors is met,
we expect the Canary to be halted until the success rate increases to 99% (set in `canary.yaml`). We then expect the 
promotion to halt, and for all traffic to be routed back to the stable version of your application.

If you would like to run through this demo on your own, please take a look at Workshop 5, [Accelerating 
the Software Development Lifecycle](https://weaveworks-gitops.awsworkshop.io/50_workshop_5_accelerating_sdlc.html).
