
## Links below: 
Big Bank Link: [where to get the charts](https://repo1.dso.mil/big-bang/bigbang) 
OR [click on the gitlaB icon](https://docs-bigbang.dso.mil/latest/docs/developer/package-integration/helm-standards/)

## Get the chart repos
- Click on charts --> Big Bank Link: [where to get the charts](https://repo1.dso.mil/big-bang/bigbang)
- click on the chart --> click on the values.yaml file --> [ctr+f] to find the repo of the application:
- Link: [values.yaml Link:](https://repo1.dso.mil/big-bang/bigbang/-/blob/master/chart/values.yaml)
- Then go put that repo in the url
- example, get the istio-system repo as such: [https://repo1.dso.mil/big-bang/product/packages/istio-system.git]

## After creating account, signout and login again
https://registry1.dso.mil/account/sign-in?redirect_url=%2Fharbor%2Fprojects    --> LOGIN VIA LOCAL DB

--> will require you to add a username for bigbank project. Mine, I added: 

## Go to your name, click on your profile and obtain a token CLI-secret:

CLI secret  C   --> 

## Create namespace istio-system , reconfirm this

```sh
kubectl create ns istio-operator
kubectl create ns istio-system 
```

## create a secret of your token-CLI secret

***HOTE:***  On this line --> kubectl -n istio-system create secret docker-registry private-registry , this name: [ private-registry ] signifies name of your [pull secret]

![pull secret](ironbank-sonar\pill-secret.png)

```sh
kubectl create secret docker-registry private-registry \
  --docker-server=registry1.dso.mil \
  --docker-username=cafanwii \
  --docker-password=zRdxM6Dy3YJCW22UzCuOERrsTJc4yzxJ \
  --docker-email=sosotech2000@gmail.com \
  --dry-run=client -o yaml > private-registry.yaml
```

## Apply the secrets to both namespaces
```sh
kubectl -n istio-operator apply -f private-registry.yaml
kubectl -n istio-system apply -f private-registry.yaml
k get secret -n istio-system
k get secret -n istio-operator
```

## clone the chart istio-operator then istio-controlplane chart

```sh
git clone  https://repo1.dso.mil/big-bang/product/packages/istio-operator.git
git clone https://repo1.dso.mil/big-bang/product/packages/istio-controlplane.git
```

## nexT: Cd into the chart and install Istio
- NOTE , CD and rename the charts as  [istio-operator] and [istio-controlplane]
- Next: delete all the other stuff that was in the charts 
- Next move [istio-operator] and [istio-controlplane] ../ to be the main directorues

```sh
mv istio-operator io
cd io 
mv chart istio-operator
mv istio-operator ../
cd ..
rm -rf io
```

```sh
mv istio-controlplane ic
cd ic
mv chart istio-controlplane
mv istio-controlplane ../
cd ..
rm -rf ic
```

## edit the values.yaml file 
Update the pull-secret with created registry value


AS SUCH:
<!-- 
imagePullSecrets: 
- private-registry 
-->

### annotate and label the istio-operator ns 
```sh
k label ns istio-operator app.kubernetes.io/managed-by=Helm
k annotate ns istio-operator meta.helm.sh/release-name=istio-operator
k label ns istio-operator meta.helm.sh/release-name=istio-operator
k annotate ns istio-operator meta.helm.sh/release-namespace="istio-operator"

k get ns istio-operator -o yaml
```

### install istio-operator
```sh
cd istio-bigbang
helm -n istio-operator install istio-operator istio-operator/
k -n istio-operator get po 
k -n istio-operator get svc
```

### install istio-system
```sh
helm -n istio-system install istio-ingress istio-controlplane/
k -n istio-system get po 
k -n istio-system get svc
# helm -n istio-system upgrade istio-ingress  istio-controlplane/
```


https://repo1.dso.mil/search?nav_source=navbar&repository_ref=main&search=proxyv2&search_code=true
https://registry1.dso.mil/harbor/projects/3/repositories

## install envoy

```sh
mv envoy-proxy io
cd io 
mv chart envoy-proxy
mv envoy-proxy ../
cd ..
rm -rf io
```

```sh
helm -n istio-system install envoy envoy-proxy/
```
 
<!-- ## Create secret for gateway istio-ingress

```sh
sudo kubectl create secret tls wildcard-cert --cert=/etc/letsencrypt/live/s3gis.be/fullchain.pem --key=/etc/letsencrypt/live/s3gis.be/privkey.pem -n istio-system --dry-run=client -o yaml > wildcard-cert-secret.yaml

k -n istio-system  apply -f wildcard-cert-secret.yaml -->
```


https://repo1.dso.mil/big-bang/bigbang
https://kubernetes.io/docs/reference/network

####------------------------------------------------------

## Mutual TLS (mTLS)
Mutual TLS (mTLS) is a security feature in Istio that enables two-way authentication between services within a service mesh, ensuring that both client and server can verify each other's identity. Here’s an overview of how mTLS works and the steps to configure it properly:

1. Understanding mTLS in Istio
In standard TLS (one-way TLS), only the server presents a certificate to verify its identity. The client trusts this certificate and initiates a secure connection.
In mutual TLS, both the client and server present certificates to verify each other's identities. This mutual authentication ensures only authorized services can communicate with each other within the mesh.
Istio’s control plane (specifically Istiod) manages the certificate distribution to services, making it easier to enforce secure communication without manually creating certificates for each service.
2. Configuring mTLS in Your Istio Mesh
Since Istio is installed with the STRICT mode for mtls.mode in your YAML file, the mesh is set to allow only encrypted and authenticated (mutually verified) traffic. You don't necessarily need to create your own certificates manually; instead, Istio can handle this automatically.

3. Enabling mTLS Globally
To fully enforce mTLS across your Istio mesh, you need to configure a PeerAuthentication policy at the mesh level or namespace level.

Create a YAML configuration to enable mTLS globally. Save this file as peer-authentication.yaml:

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

This configuration tells Istio to enforce mTLS in the entire mesh, ensuring all services can only communicate with each other using authenticated TLS connections.

4. Certificates in Istio
By default, Istio uses Citadel (an internal certificate authority in Istiod) to automatically issue and manage certificates for services. The control plane handles certificate rotation, distribution, and expiration automatically.

If you need custom certificates for specific requirements (e.g., compliance or integration with an external CA), you can configure an external CA, but this is optional for basic mTLS setup.

5. Verification Steps
After configuring mTLS:

Check that services within the mesh communicate only via mTLS by inspecting traffic between services.
You can also use tools like Kiali (if enabled) to visualize and monitor mTLS connections within the mesh.
6. Testing mTLS
To test if mTLS is working:

Deploy two sample services in your Istio mesh.
Check the logs of the Istio sidecar containers for both services to verify that the traffic is encrypted and authenticated.
By enabling mTLS with the STRICT mode and applying a PeerAuthentication policy, you’re setting up robust, encrypted communication for all services within your Istio mesh.

### OPTIONALLY, If you want to enforce mutual TLS (mTLS) for all namespaces in your Istio service mesh, 
you don’t need to create separate PeerAuthentication policies for each namespace. You can apply a single PeerAuthentication policy at the mesh level (usually in the istio-system namespace) with mtls.mode: STRICT, and it will propagate across all namespaces by default.

How It Works:
A PeerAuthentication policy with mtls.mode: STRICT applied at the mesh level configures Istio to enforce mTLS for all traffic within the entire mesh, including services across multiple namespaces.
This global configuration ensures consistent security settings across namespaces without having to define separate policies.
Optional Namespace-Specific Policies
If you want more control—for example, if you want to allow plaintext (non-mTLS) traffic in some namespaces but enforce mTLS in others—you can create namespace-specific PeerAuthentication policies.

Namespace-level policy: A PeerAuthentication policy applied at the namespace level will override the mesh-level configuration for services in that specific namespace.
Service-level policy: You can also define PeerAuthentication policies for individual services within a namespace if finer control is needed.
Example Scenario
Suppose you want the following setup:

Enforce mTLS globally, except for a specific namespace (test-namespace) where you allow both mTLS and plaintext traffic.
You can achieve this with two configurations:

- Mesh-level policy (Strict mTLS for all):

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

Namespace-specific policy (Allow both mTLS and plaintext in test-namespace). In this setup,
All namespaces will have STRICT mTLS, except test-namespace, where both mTLS and non-mTLS traffic are allowed due to the PERMISSIVE mode:

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: test-namespace-policy
  namespace: test-namespace
spec:
  mtls:
    mode: PERMISSIVE
```

## configuring serviceannotations in ingressgateway

[Service-annotations](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.9/guide/service/annotations/)