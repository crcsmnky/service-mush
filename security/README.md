# Debugging Istio Security Deployments

## Deploy the Weather App

Before deploying the sample [weatherinfo](https://github.com/crcsmnky/weatherinfo) app, you'll need an API key from [OpenWeatherMap](http://openweathermap.org/api). Once you have that, use it to create a Secret:
- `kubectl create secret generic openweathermap --from-literal=apikey=[OPENWEATHERMAP-API-KEY]`

Next, create the Istio Gateway, VirtualService, DestinationRule, and ServiceEntry objects needed for the sample app to function properly:
- `kubectl apply -f weatherapp-all.yaml`

```
stern weather-frontend -c istio-proxy -e ".*/source.*"

stern weather-backend-multiple -c istio-proxy -e ".*source/"
```

## Step 1 -- Apply the Policy + Dest. Rule 

*show frontend working* 

```
apply mtls-backend-1.yaml 
``` 

## Step 2 -- Show how far traffic gets 

*show stern logs* 


## Step 3 -- Control plane logs? (show healthy)

PILOT: 
```
stern -n istio-system pilot -c discovery
```

CITADEL:
```
kubectl get deploy -l istio=citadel -n istio-system
``` 

## Step 4 -- Do the envoy sidecars have certs, and are they valid / not expired?  

*remember that mTLS means that both the frontend and backend must have valid certs* 
*client-side (frontend)* 

```
 kubectl exec $(kubectl get pod -l app=weather-frontend -o jsonpath={.items..metadata.name}) -c istio-proxy -- cat /etc/certs/cert-chain.pem | openssl x509 -text -noout  | grep Validity -A 2
``` 

```
 kubectl exec weather-backend-multiple-557bbc75bb-wbrwh -c istio-proxy -- cat /etc/certs/cert-chain.pem | openssl x509 -text -noout  | grep Validity -A 2
```

## Step 5 -- Enable trace-level Envoy logs for the backend 

*we applied the Policy for the backend, so that might be the most likely culprit* 

```
BACKEND_POD=$(kubectl get pod  | grep weather-backend-multiple | awk '{ print $1 }'); kubectl exec -it $BACKEND_POD -c istio-proxy /bin/sh 

curl -X POST "http://localhost:15000/logging?level=trace"
```

*stern: show debug logs streaming in*  

## Step 6 -- refresh the frontend more, and see the error 

*hard to visually grep -- exit stern and do an actual grep* 

```
kubectl logs $BACKEND_POD -c istio-proxy | grep handshake
```
*show handshake error = 1* 

This tells us that the initial TLS handshake between frontend/backend failed. Something went wrong in the initial exchange of keys and certs... 

By powers of deduction we've checked that 
- The istio control plane is healthy / sending certs to pods 
- Both the envoy sidecars have valid certs and keys

At this point I'll assume my YAML is off, or something on the configuration side has gone wrong. 


## Step 7 -- Diagnose the Configuration Error to find the problem 


```
istioctl authn tls-check weather-backend.default.svc.cluster.local 
```

*client conflict means that clients OF weather-backend aren't configured for TLS* 

*what's responsible for this half of the mTLS connection? destinationrule.* 

```
kubectl get dr 
```


## Fix 

```
delete mtls-backend-1.yaml 

apply mtls-backend-2.yaml  -- which updates the weather-backend destinationrule to enforce mTLS 
```

```
istioctl authn tls-check weather-backend.default.svc.cluster.local 
```


