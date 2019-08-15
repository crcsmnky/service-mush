# Service Mush: Debugging Istio Deployments

## Cluster Setup

Create a GKE cluster:
- `gcloud container clusters create debugging-istio --cluster-version=latest --machine-type=n1-standard-2`

Grab the cluster credentials:
- `gcloud container clusters get-credentials debugging-istio`

Create `cluster-admin` role binding:
- `kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value core/account)`

Download and unpack Istio:
- `curl -L https://git.io/getLatestIstio | sh -`
- `cd istio-1.2.4`

Add `istioctl` to your path:
- `export PATH=$PWD/bin:$PATH`

Install Istio:
- `kubectl apply -f install/kubernetest/istio-demo.yaml`

Wait for the Istio control plane components to be installed and ready:
- `kubectl get pod -n istio-system -w`

Enable auto-injection for `istio-proxy`:
- `kubectl label ns default istio-injection=enabled`

## Debugging Topics

Refer to the individual sections below for debugging deep-dives:
- [Traffic](traffic/)
- [Telemetry](telemetry/)
- [Security (mTLS)](security/)