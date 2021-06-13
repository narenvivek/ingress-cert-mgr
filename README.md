# Kubernetes NGINX Ingress Controller and TLS certs (LetsCncrypt)
This guide assumes that you have already created a K8s cluster somethere. I have used Azure Kubernetes Services but this should work for any public K8s cluster deployment. 

The exceptions are:

* Local K8s cluster that you might have deployed with your `Docker Desktop` will NOT work
* Cert Manager portion of the description below will also NOT work for private clusters (E.g.: clusters behind Azure Private Endpoint or behind Azure App Gateway that already has an edge cert deployed)

You will need the following:

1. Deployed cluster that has a public IP issued
2. Kube context is set to use the correct cluster (`kubectl config get-contexts` should tell you the cluster you are using)
3. You are authenticated to and have required permissions to Kubernetes APIs exposed by the cluster
4. You are able to create a DNS zone record (A-record) to point to the public cluster service IP address

Once these are in place, following steps will guide you through - deployment of NGINX Ingress Controller, sample application in the cluster (with a service that exposes it) and usage of `cert-manager` to use proper TLS certificate issued by LetsEncrypt.

# Deploy the NGINX Ingress Controller

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

helm repo update

helm install ing-ctrl ingress-nginx/ingress-nginx

kubectl get svc

NAME                                          TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                      AGE
ing-ctrl-ingress-nginx-controller             LoadBalancer   10.0.204.18    20.85.33.145   80:31169/TCP,443:31250/TCP   47m
ing-ctrl-ingress-nginx-controller-admission   ClusterIP      10.0.189.222   <none>         443/TCP                      47m
kubernetes                                    ClusterIP      10.0.0.1       <none>         443/TCP                      50m

```

# Assign a DNS name
Use Azure Zone to create an A-record to point to Azure Load Balancer. A-record points to 20.85.33.145 in our case.

# Deploy an Example App, Service (ClusterIP) and Ingress Route to the app
Review `app.yml` and `service.yml` which is the application and service part.

Ingress route without TLS is `ingress.yml`

Deploy with a single command:

`kubectl apply -f app.yml -f service.yml -f ingress.yml`

Ensure your app is accessible using the FQDN and test. You will get a cert error if you are using HTTPS since this is the fake certificate that NGINX Ingress Controller would deploy automatically.

# Let's Encrypt - Deploy Cert Manager

Cert Manager MUST be created in its own namespace

```
kubectl create namespace cert-manager

helm repo add jetstack https://charts.jetstack.io

helm repo update

helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.3.1
```

## Verify cert-manager installation

`kubectl get pods --namespace cert-manager`

## Configure LetsEncypt issuer (one for Staging and one for Prod)

```
# Staging
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # The ACME server URL
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: test@gmail.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-staging
    # Enable the HTTP-01 challenge provider
    solvers:
      - http01:
          ingress:
            class: nginx
```

Deploy Staging Issuer manifest:

`kubectl apply -f issuer-staging.yml`

## Verify Issuer deployment and registration with LetsEncrypt

`kubectl describe issuer letsencrypt-staging`

You should see something like below:

```
  Conditions:
    Last Transition Time:  2021-06-13T00:28:05Z
    Message:               The ACME account was registered with the ACME server
    Observed Generation:   1
    Reason:                ACMEAccountRegistered
    Status:                True
    Type:                  Ready
```

Now, your issuer is correctly registered to LetsEncrypt

## Deploy a TLS Ingress Resource
This is just a few lines changes to `ingress.yml` to add `tls` portion of YAML and an addition of `cert-manager` annotation. Review `ingress-tls-staging.yml` and compare it with `ingress.yml`.

Adding `cert-manager` annotation:

`cert-manager.io/issuer: "letsencrypt-staging"`

Changes to `rules` list:

```
  tls:
  - hosts:
    - azurekubed.com
    secretName: quickstart-example-tls # you can choose whatever secretName you want
```

Deploy YAML manifest:

`kubectl apply -f ingress-staging.yml`

After you deploy LetsEncrypt takes about 2-5 mins to obtain the certs (tls.crt and tls.key). You can verify if the certs have come through using:

`kubectl get certificate`

You will see that the `Ready` column will be `False` to start with. Wait till it becomes true. If you are impatient and want to see the events use:

`kubectl describe certificate quickstart-example-tls`

This will provide you with event details all the way through to cert issuance. When completed, you will something like this:

```
  Conditions:
    Last Transition Time:  2021-06-13T00:28:05Z
    Message:               Certificate is up to date and has not expired
    Observed Generation:   1
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2021-09-10T23:26:16Z
  Not Before:              2021-06-12T23:26:17Z
  Renewal Time:            2021-08-11T23:26:16Z
```

In order to confirm that the `cert-manager` has downloaded the crypto primitives, you could check the status of Kube `secret`:

`kubectl describe secret quickstart-example-tls`

You should see something like this:

```
Name:         quickstart-example-tls
Namespace:    default
Labels:       <none>
Annotations:  cert-manager.io/alt-names: azurekubed.com
              cert-manager.io/certificate-name: quickstart-example-tls
              cert-manager.io/common-name: azurekubed.com
              cert-manager.io/ip-sans:
              cert-manager.io/issuer-group: cert-manager.io
              cert-manager.io/issuer-kind: Issuer
              cert-manager.io/issuer-name: letsencrypt-staging
              cert-manager.io/uri-sans:

Type:  kubernetes.io/tls

Data
====
tls.crt:  5587 bytes
tls.key:  1679 bytes
```

The presence of `tls.crt` and `tls.key` confirms that the cert is issued correctly.

# Clean-up

## Delete cluster artifacts

`kubectl delete -f kube-manifests\01-App -f kube-manifests\02-Cert-Manager-Issuer -f kube-manifests\03-Ingress`

## Delete Ingress controller and cert-manager using Helm

```
helm uninstall ing-ctrl
helm uninstall cert-manager
```

## Delete Cluster (Azure)
`az aks delete --name <cluster-name> --resource-group <resource-group-name>`
`az group delete --resource-group <resource-group-name>`