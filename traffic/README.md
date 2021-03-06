# Debugging Istio Traffic Management

This repo contains a demo/tutorial for debugging Istio traffic splitting. 

## Deploy the Weather App

Before deploying the sample [weatherinfo](https://github.com/crcsmnky/weatherinfo) app, you'll need an API key from [OpenWeatherMap](http://openweathermap.org/api). Once you have that, use it to create a Secret:
- `kubectl create secret generic openweathermap --from-literal=apikey=[OPENWEATHERMAP-API-KEY]`

Next, create the Istio Gateway, VirtualService, DestinationRule, and ServiceEntry objects needed for the sample app to function properly:
- `kubectl apply -f weather-rules.yaml`
- `kubectl apply -f weather-deployment.yaml`

Now access the app using the `istio-ingressgateway` Load Balancer IP address:
* `INGRESSGATEWAY=$(kubectl get svc -n istio-system istio-ingressgateway -o jsonpath="{.status.loadBalancer.ingress..ip}")`
* Open `http://$INGRESSGATEWAY` in a browser

Hit **Refresh** a few times and you'll notice that you're seeing the frontend populated by two different backends at about a 50/50 split. According to `weather-rules.yaml` you should be seeing a 90/10 split between `version: single` and `version: multiple`. So now you can assume that Istio routing rules aren't being properly applied since standard Kubernetes routing uses round-robin (hence 50/50 for the two services).

## Debug the Weather App

Start by checking Pods and Services and confirm everything is running:
- `kubectl get pods`
- `kubectl get service`

Next, check that rules actually exist:
- `kubectl get virtualservice`

Now deploy the [`respy`](https://github.com/askmeegs/respy) Pod and use it to check the distribution percentage across versions of `weather-backend`:
- `kubectl apply -f respy.yaml`
- `RESPY=$(kubectl get pod -l app=respy -o jsonpath="{.items[0].metadata.name}")`
- `kubectl exec -it $RESPY -c respy -- ./respy --u http://weather-backend:5000/version`
```
☎️   1000 requests to http://weather-backend:5000/version...
+--------------------------------+--------------------+
|            RESPONSE            | % OF 1000 REQUESTS |
+--------------------------------+--------------------+
| weather-backend: single        | 50%                |
| weather-backend: multiple      | 50%                |
+--------------------------------+--------------------+
```

As you can see, after 1000 requests, traffic to `weather-backend` is still using the 50/50 split.

Now check the `istio-proxy` log from `weather-frontend`:
- `FRONTEND=$(kubectl get po -l app=weather-frontend -o jsonpath="{.items..metadata.name}")`
- `kubectl logs $FRONTEND -c istio-proxy -f | grep 'weather-backend'`

In the log, you'll see entries like the following:
```
[2019-04-05 20:03:16.038][18][info][upstream] external/envoy/source/common/upstream/cluster_manager_impl.cc:494] add/update cluster outbound|5000||weather-backend.default.svc.cluster.local during init
[2019-04-05 20:03:16.039][18][info][upstream] external/envoy/source/common/upstream/cluster_manager_impl.cc:494] add/update cluster outbound|5000|single|weather-backend.default.svc.cluster.local during init
[2019-04-05 20:03:16.040][18][info][upstream] external/envoy/source/common/upstream/cluster_manager_impl.cc:494] add/update cluster outbound|5000|multiple|weather-backend.default.svc.cluster.local during init
[2019-04-05T20:04:13.272Z] - 162 1009 270 "10.52.3.3:5000" outbound|5000||weather-backend.default.svc.cluster.local 10.52.3.2:41068 10.55.255.116:5000 10.52.3.2:44988
```

Note that `weather-frontend` is able to connect to `weather-backend`, but the traffic is not directed to any particular version. So we've now confirmed that routing rules aren't being used and `weather-frontend` is using Kubernetes round-robin behavior to route traffic.

Now, check that routing rules have been synced to `istio-proxy` Pods:
- `istioctl proxy-status`
```
NAME                                                  CDS        LDS        EDS               RDS          PILOT                            VERSION
istio-egressgateway-f7b8cc667-69r9k.istio-system      SYNCED     SYNCED     SYNCED (100%)     NOT SENT     istio-pilot-68f587df5d-r4vhn     1.2.2
istio-ingressgateway-cfbf989b7-b8kt9.istio-system     SYNCED     SYNCED     SYNCED (100%)     SYNCED       istio-pilot-68f587df5d-r4vhn     1.2.2
respy-5b979cd985-vh548.default                        SYNCED     SYNCED     SYNCED (100%)     SYNCED       istio-pilot-68f587df5d-57ll9     1.2.2
sleep-7674d45776-fsjm7.default                        SYNCED     SYNCED     SYNCED (100%)     SYNCED       istio-pilot-68f587df5d-r4vhn     1.2.2
weather-backend-multiple-7bb94f788c-zthzz.default     SYNCED     SYNCED     SYNCED (50%)      SYNCED       istio-pilot-68f587df5d-r4vhn     1.2.2
weather-backend-single-865c7779c9-lq9kh.default       SYNCED     SYNCED     SYNCED (50%)      SYNCED       istio-pilot-68f587df5d-57ll9     1.2.2
weather-frontend-565cdbbb67-l9c8f.default             SYNCED     SYNCED     SYNCED (50%)      SYNCED       istio-pilot-68f587df5d-57ll9     1.2.2
```
- `istioctl proxy-status $FRONTEND.default`
```
Clusters Match
Listeners Match
Routes Match
```

Next, check `istio-proxy` configuration on `weather-frontend`, starting with `cluster` configuration:
- `istioctl proxy-config cluster $FRONTEND`
```
SERVICE FQDN                                              PORT      SUBSET       DIRECTION     TYPE
*.googleapis.com                                          80        -            outbound      ORIGINAL_DST
*.googleapis.com                                          443       -            outbound      ORIGINAL_DST
BlackHoleCluster                                          -         -            -             STATIC
accounts.google.com                                       80        -            outbound      ORIGINAL_DST
accounts.google.com                                       443       -            outbound      ORIGINAL_DST
api.openweathermap.org                                    80        -            outbound      ORIGINAL_DST
heapster.kube-system.svc.cluster.local                    80        -            outbound      EDS
istio-citadel.istio-system.svc.cluster.local              8060      -            outbound      EDS
istio-citadel.istio-system.svc.cluster.local              9093      -            outbound      EDS
istio-egressgateway.istio-system.svc.cluster.local        80        -            outbound      EDS
istio-egressgateway.istio-system.svc.cluster.local        443       -            outbound      EDS
istio-galley.istio-system.svc.cluster.local               443       -            outbound      EDS
istio-galley.istio-system.svc.cluster.local               9093      -            outbound      EDS
istio-ingressgateway.istio-system.svc.cluster.local       80        -            outbound      EDS
istio-ingressgateway.istio-system.svc.cluster.local       443       -            outbound      EDS
istio-ingressgateway.istio-system.svc.cluster.local       853       -            outbound      EDS
istio-ingressgateway.istio-system.svc.cluster.local       8060      -            outbound      EDS
istio-ingressgateway.istio-system.svc.cluster.local       15011     -            outbound      EDS
istio-ingressgateway.istio-system.svc.cluster.local       15030     -            outbound      EDS
istio-ingressgateway.istio-system.svc.cluster.local       15031     -            outbound      EDS
istio-ingressgateway.istio-system.svc.cluster.local       31400     -            outbound      EDS
istio-pilot.istio-system.svc.cluster.local                8080      -            outbound      EDS
istio-pilot.istio-system.svc.cluster.local                9093      -            outbound      EDS
istio-pilot.istio-system.svc.cluster.local                15010     -            outbound      EDS
istio-pilot.istio-system.svc.cluster.local                15011     -            outbound      EDS
istio-policy.istio-system.svc.cluster.local               9091      -            outbound      EDS
istio-policy.istio-system.svc.cluster.local               9093      -            outbound      EDS
istio-policy.istio-system.svc.cluster.local               15004     -            outbound      EDS
istio-sidecar-injector.istio-system.svc.cluster.local     443       -            outbound      EDS
istio-telemetry.istio-system.svc.cluster.local            9091      -            outbound      EDS
istio-telemetry.istio-system.svc.cluster.local            9093      -            outbound      EDS
istio-telemetry.istio-system.svc.cluster.local            15004     -            outbound      EDS
istio-telemetry.istio-system.svc.cluster.local            42422     -            outbound      EDS
kube-dns.kube-system.svc.cluster.local                    53        -            outbound      EDS
kubernetes.default.svc.cluster.local                      443       -            outbound      EDS
metadata.google.internal                                  80        -            outbound      ORIGINAL_DST
metadata.google.internal                                  443       -            outbound      ORIGINAL_DST
metrics-server.kube-system.svc.cluster.local              443       -            outbound      EDS
prometheus_stats                                          -         -            -             STATIC
promsd.istio-system.svc.cluster.local                     9090      -            outbound      EDS
weather-backend.default.svc.cluster.local                 5000      -            outbound      EDS
weather-backend.default.svc.cluster.local                 5000      multiple     outbound      EDS
weather-backend.default.svc.cluster.local                 5000      single       outbound      EDS
weather-frontend.default.svc.cluster.local                5000      -            inbound       STATIC
weather-frontend.default.svc.cluster.local                5000      -            outbound      EDS
xds-grpc                                                  -         -            -             STRICT_DNS
zipkin                                                    -         -            -             STRICT_DNS
```

Everything looks good, now let's check the outbound routes from `istio-proxy`:
- `istioctl proxy-config routes $FRONTEND`
```
NOTE: This output only contains routes loaded via RDS.
NAME                                                         VIRTUAL HOSTS
80                                                           7
5000                                                         1
8060                                                         1
8080                                                         1
9090                                                         1
9091                                                         2
9093                                                         5
15004                                                        2
15010                                                        1
15030                                                        1
15031                                                        1
inbound|5000||weather-frontend.default.svc.cluster.local     1
inbound|5000||weather-frontend.default.svc.cluster.local     1
```

Now we know that `weather-backend` is running on port `5000` so let's dig into that route:
- `istioctl proxy-config routes $FRONTEND --name 5000 -o json`
```
[
    {
        "name": "5000",
        "virtualHosts": [
            {
                "name": "weather-frontend.default.svc.cluster.local:5000",
                "domains": [
                    "weather-frontend.default.svc.cluster.local",
                    "weather-frontend.default.svc.cluster.local:5000",
                    "weather-frontend",
                    "weather-frontend:5000",
                    "weather-frontend.default.svc.cluster",
                    "weather-frontend.default.svc.cluster:5000",
                    "weather-frontend.default.svc",
                    "weather-frontend.default.svc:5000",
                    "weather-frontend.default",
                    "weather-frontend.default:5000",
                    "10.59.245.6",
                    "10.59.245.6:5000"
                ],
                "routes": [
                    {
                        "match": {
                            "prefix": "/"
                        },
                        "route": {
                            "cluster": "outbound|5000||weather-frontend.default.svc.cluster.local",
                            "timeout": "0.000s",
                            "maxGrpcTimeout": "0.000s"
                        },
                        "decorator": {
                            "operation": "weather-frontend.default.svc.cluster.local:5000/*"
                        },
                        "perFilterConfig": {
                            "mixer": {
                                "disable_check_calls": true,
                                "forward_attributes": {
                                    "attributes": {
                                        "destination.service": {
                                            "string_value": "weather-frontend.default.svc.cluster.local"
                                        },
                                        "destination.service.host": {
                                            "string_value": "weather-frontend.default.svc.cluster.local"
                                        },
                                        "destination.service.name": {
                                            "string_value": "weather-frontend"
                                        },
                                        "destination.service.namespace": {
                                            "string_value": "default"
                                        },
                                        "destination.service.uid": {
                                            "string_value": "istio://default/services/weather-frontend"
                                        }
                                    }
                                },
                                "mixer_attributes": {
                                    "attributes": {
                                        "destination.service": {
                                            "string_value": "weather-frontend.default.svc.cluster.local"
                                        },
                                        "destination.service.host": {
                                            "string_value": "weather-frontend.default.svc.cluster.local"
                                        },
                                        "destination.service.name": {
                                            "string_value": "weather-frontend"
                                        },
                                        "destination.service.namespace": {
                                            "string_value": "default"
                                        },
                                        "destination.service.uid": {
                                            "string_value": "istio://default/services/weather-frontend"
                                        }
                                    }
                                }
                            }
                        }
                    }
                ]
            }
        ],
        "validateClusters": false
    }
]
```

*Note*: the output of `istioctl proxy-config routes` can be lengthy and difficult to parse. Consider using [`jq`](https://stedolan.github.io/jq/) to simplify visualizing the output. For example: `istioctl proxy-config routes $FRONTEND --name 5000 -o json | jq '.[].virtualHosts[]|.name,.domains'` will show each route's name and associated domains.

The only route on port `5000` that `istio-proxy` knows about is back to `weather-frontend`. So Istio doesn't have any specific routes for `weather-backend`.

Beyond including `istio-proxy` in your Pods, the only way for Services to be available for routing is through the Service definition. So let's take a look at that:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: weather-backend
  labels:
    app: weather-backend
spec:
  ports:
  - port: 5000
    name: backend
  selector:
    app: weather-backend
```

And there is the culprit: `spec.ports[0].name` is set to `backend`. For Istio's routing rules to apply, [Service port names must follow a specific naming convention](https://istio.io/docs/setup/kubernetes/prepare/requirements/).

So now, update the Service definition:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: weather-backend
  labels:
    app: weather-backend
spec:
  ports:
  - port: 5000
    name: http-backend
  selector:
    app: weather-backend
EOF
```

Once that's applied, wait a few moments and check the rules syncing status to confirm the changes have been applied:
- `istioctl proxy-status`

Then you can also check the `istio-proxy` routes in the `weather-frontend` Pod:
- `istioctl proxy-config route $FRONTEND --name 5000 -o json`

And now you see that the `istio-proxy` has route entries for both `weather-backend` subsets.
