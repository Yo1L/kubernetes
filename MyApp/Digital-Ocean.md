# Introduction

Digital Ocean is an affordable alternative to GCloud and AWS supporting k8s.

## New Cluster

We need some extra setup to make it work :

- Setup Helm
- Setup Ingress (Reverse Proxy)
- Setup CertManager to compute your Let's Encrypt HTTPS certficates
- Setup Redis (Sessions and Cache)
- Setup Gitlab-runner
- Setup Database
- Setup Gitab registry token

### Setup HELM

First, we need to authorize tiller (helm pod) before to avoid running problems with your packages.

Here the steps to install Helm:
```
$ kubectl create -f ./rbac-tiller.yml
```

Init helm
```
$ helm init --service-account tiller
```

#### Remove an existing helm
If you already have installed helm, here the steps to remove it :
```
$ helm reset # Only if helm is already installed
Tiller (the Helm server-side component) has been uninstalled from your Kubernetes Cluster.

$ kubectl delete -f ./rbac-tiller.yml # Only if helm is already installed
serviceaccount "tiller" deleted
clusterrolebinding "tiller-clusterrolebinding" deleted
```

If you have some trouble during the uninstall, try :
 - To find the tiller pod name :
```
kubectl get pods -n kube-system | grep tiller
```
 - To remove the pod by hand (if there is one):
```
kubect delete pod [TILLER POD]
```

### Setup Ingress
A reverse proxy permits to link between your URL and your HTTP/HTTPS services

To install [stable/nginx-ingress](https://github.com/helm/charts/tree/master/stable/nginx-ingress) with at least 1 replica per node :
```
helm install -f values.yaml --name nginx-ingress stable/nginx-ingress
```

To get the external ip 
```
kubectl -n default get services -o wide -w nginx-ingress-controller
```

### Setup Redis

We will using [stable/redis](https://github.com/helm/charts/tree/master/stable/redis) helm package

Easy peasy :
```
helm install --name cache stable/redis --set cluster.enabled=false,cluster.slaveCount=0,master.persistence.enabled=false
```

To retrieve your password :
```
export REDIS_PASSWORD=$(kubectl get secret --namespace default cache-redis -o jsonpath="{.data.redis-password}" | base64 --decode)
```

Modify your Laravel .env accordingly:
```
REDIS_HOST=rolling-bumblebee-redis-master
REDIS_PASSWORD=[PASSWORD HERE]
```

### Setup CertManager

Before starting, be sure to reference your global static ip at your DNS provider (A Entry).

Setup:
```
helm install \
    --name cert-manager \
    --namespace kube-system \
    stable/cert-manager
```

#### How it works
We will issue an ACME certificate using HTTP validation (http01 which is automatic, no dns manual modification).

Here is a issuer sample in staging mode (url) to avoid blacklisting in case of to many errors/tries:
```
apiVersion: certmanager.k8s.io/v1alpha1
kind: Issuer
metadata:
  name: letsencrypt-issuer-staging
  namespace: default
spec:
  acme:
    # The ACME server URL
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: contact@example.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-example-secret
    # Enable the HTTP-01 challenge provider
    http01: {}
```

Apply this issuer configuration:
```
kubectl apply -f issuer.yaml
```

Prepare your certificate request:
```
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: myapp-cert
  namespace: default
spec:
  secretName: letsencrypt-issuer-staging
  issuerRef:
    name: myapp-issuer
  commonName: myapp.com
  dnsNames:
  - www.myapp.com
  acme:
    config:
    - http01:
        ingress: myapp-ingress
      domains:
      - myapp.com
    - http01:
        ingress: myapp-ingress
      domains:
      - www.myapp.com
```

Apply this certificate configuration:
```
kubectl apply -f certificate.yaml
```

Wait for your validation:
```
kubectl describe certificate myapp-cert
...

Events:
  Type    Reason          Age   From          Message
  ----    ------          ----  ----          -------
  Normal  CreateOrder     12m   cert-manager  Created new ACME order, attempting validation...
  Normal  DomainVerified  4m    cert-manager  Domain "myapp.com" verified with "http-01" validation
  Normal  DomainVerified  4m    cert-manager  Domain "www.myapp.com" verified with "http-01" validation
  Normal  IssueCert       3m    cert-manager  Issuing certificate...
  Normal  CertObtained    3m    cert-manager  Obtained certificate from ACME server
  Normal  CertIssued      3m    cert-manager  Certificate issued successfully
```

Check the certificate:
```
kubectl get secret myapp-tls -o yaml
...
apiVersion: v1
data:
  tls.crt: 
  ...
  tls.key:
  ...
kind: Secret
metadata:
  annotations:
    certmanager.k8s.io/alt-names: myapp.com,www.myapp.com
    certmanager.k8s.io/common-name: myapp.com
    certmanager.k8s.io/issuer-kind: Issuer
    certmanager.k8s.io/issuer-name: myapp-issuer
  creationTimestamp: 2018-08-10T08:46:23Z
  labels:
    certmanager.k8s.io/certificate-name: myapp-cert
  name: myapp-tls
  namespace: default
  resourceVersion: "xxxxx"
  selfLink: /api/v1/namespaces/default/secrets/myapp-tls
  uid: xxxxxxxxxx-xxxx-xxxx-xxxxxxxxxx
type: kubernetes.io/tls
```

Now, if everything works we can ask for a real certificate (not staging). To do so, we just need to change the url in our issuer configuration to:

#### Problems with your validation ?

Check for any mispelled DNS or Ingress labels in your configurations. 

Check whether there are new spec.rules in your referenced ingress (myapp-ingress):
```
kubectl get ing
```
or 
```
kubect describe ing myapp-ingress
```

#### Modify your ingress to support SSL

Just add the TLS configuration in your ingress configuration under spec:
```
spec:
  tls:
  - hosts:
    - myapp.com
    secretName: myapp-tls
```

A Complete sample:
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.global-static-ip-name: "cluster-static-ip"
    nginx.ingress.kubernetes.io/rewrite-target: /
  name: myapp-ingress
spec:
  tls:
  - hosts:
    - myapp.com
    secretName: myapp-tls
  backend:
    serviceName: myapp-web-svc
    servicePort: 80
  rules:
    - host: myapp.com
      http:
        paths:
          - backend:
              serviceName: myapp-web-svc
              servicePort: 80
```

#### Forcing certificate renewal

You just need to delete the tls secret:
```
kubectl delete secret myapp-tls
```

#### Monitoring certification creation/renewal

Check events of your ingress rules
```
kubectl describe ing
```

Check events of your certificates requests:
```
kubectl describe certificate
```

#### Monitor certificate creation/renewal

```
kubectl describe certificate
```

##### SOURCES: 
 - [CertManager ReadTheDocs](https://cert-manager.readthedocs.io/en/latest/)
 - [HTTP Validation](https://cert-manager.readthedocs.io/en/latest/tutorials/acme/http-validation.html)   from CertManager ReadTheDocs
 - [ACME Staging Env URLs](https://letsencrypt.org/docs/staging-environment/)

### Setup Gitlab-Runner

It needs a values.yaml as explained [here](https://docs.gitlab.com/ee/install/kubernetes/gitlab_runner_chart.html).

You need to retrieve your runnerRegistrationToken from the gitlab.com runners configuration page in order to update your values.yaml.

Then add gitlab repo in helm :
```
helm repo add gitlab https://charts.gitlab.io
```

In the directory containing your values.yaml
```
helm install --name gitlab-runner -f ./values.yaml gitlab/gitlab-runner
```

Create a gitlab user to access your cluster for deploy purposes :
```
kubectl apply -f gitlab-user.yml
```

List secrets in the right namespace and search for a gitlab-token-XXXX (where X are replaced with random chars):
```
kubectl get secret -n gitlab-managed-apps
```

Then to get your token (replace X with the right chars):
```
kubectl describe secret -n gitlab-managed-apps gitlab-token-XXXX
```
Copy and paste only chars after "token:"

To retrieve your cluster Kubernetes master url:
```
kubectl cluster-info
```

### Setup MySQL Database

Create a volume claim for your database
```
kubectl apply -f vc.yml
```

Using the stable/mysql helm chart, go to helm/mysql:

```
helm install --name database -f values.yaml stable/mysql
```

To connect to mysql database, you'll need the pod name (tab completion or kubect get pods), then:
```
kubetctl exec -it database-mysql-xxxxxx -- mysql -p
```

### Private Docker Registy Secret for Gitlab-Runner

#### Dynamic GitLab CI secret token
Each pipeline create a secret with the temporary token to deploy so a good way is to create a secret on the fly and references it in your imagePullSecrets.


#### Static GitLab CI secret token
If you need to store the credentials of gitlab in our cluster, the secret will be labeled "myteam-gitlab" and can be used by any project.

We only need to create it a the very first initialization of the cluster (only the first time), so check whether this secret is not already set:
```
kubectl get secret
```

If there is no "myteam-gitlab", create one as follow:
```
kubectl create secret docker-registry myteam-gitlab --docker-server=registry.gitlab.com --docker-username=USERNAME --docker-password=PASSWORD --docker-email=EMAIL
```

Add in your deployment (yaml) :
```
spec:
  template:
    ...
      imagePullSecrets:
      - name: myteam-gitlab
```