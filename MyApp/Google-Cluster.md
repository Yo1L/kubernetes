# Introduction

Google Cloud offers many services like :
 - storage
 - Cloud SQL : HA MySQL Server
 - Load Balancing through Ingress rules with automatic probes generation

## New Cluster

We need some extra setup to make it work :

- Setup Helm
- Setup Ingress (Reverse Proxy or use GKE L7)
- Setup your Cloud SQL Access (your MySQL Server)
- Setup CertManager to compute your Let's Encrypt HTTPS certficates
- Setup Redis (Sessions and Cache)

### Setup HELM

In Google Cloud, we need to authorize tiller (helm pod) before to avoid running problems with your packages.

Here the steps to install Helm:
```
$ kubectl create -f ./rbac-tiller.yml
serviceaccount "tiller" created
clusterrolebinding "tiller-clusterrolebinding" created

$ helm init --service-account tiller
$HELM_HOME has been configured at ...

Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.
Happy Helming!

$ helm install "your package"
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

Two solutions here :

1. **Google Ingress LoadBalancer**

    - Pros
      - No Setup
      - No Maintenance
      - Automatic recuperation of your static ip
      - CPU Free for your Cluster

    - Cons
      - Costs

2. **NGinx Helm Ingress**

    - Pros : 
      - Really speed to changes
      - No Cost : run in your GKE Cluster
      - Tune as you want (Expert)
    - Cons :
      - Setup
      - Configuration (Static IP by hand, create the instance...)
      - Use your cluster CPU
      - Maintenance (Versions...) 

#### Use Google Ingress LoadBalancer

You just have to create an ingress configuration like :
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.global-static-ip-name: "cluster-static-ip"
    nginx.ingress.kubernetes.io/rewrite-target: /
  name: myapp-ingress
#  namespace: foo
spec:
  backend:
    serviceName: myapp-web-deployment
    servicePort: 80
  rules:
    - host: www.myapp.com
      http:
        paths:
          - backend:
              serviceName: myapp-web-deployment
              servicePort: 80
```

GCP will create a probe based on your readinessProbe/livenessProbe path, so be sure to reply a 200 HTTP Status Code on your readinessProbe/livenessProbe path or on / if none provided.

SOURCE: [Google Cloud](https://cloud.google.com/kubernetes-engine/docs/tutorials/http-balancer)

##### Custom Probe

In your Laravel app, create a simple route /health and reference it in your probes.

The /health route:
```
route::get('/health', function() {
  return 'OK';
});
```

The deployment configuration with a probe:
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: myapp-web-deployment
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: myapp-web
    spec:
      containers:
      - name: myapp-web-pod
        image: registry.gitlab.com/myteam/myapp:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: /var/www/html/.env
          name: config
          subPath: .env
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 1
          timeoutSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 1
          timeoutSeconds: 5
```

#### Setup your OWN ingress

To install [stable/nginx-ingress](https://github.com/helm/charts/tree/master/stable/nginx-ingress) with at least 2 replicas :
```
helm install stable/nginx-ingress --name nginx-ingress --set controller.replicaCount=2
```

To get the external ip 
```
kubectl -n default get services -o wide -w nginx-ingress-controller
```

### Setup your Cloud SQL Access

Access to your Cloud SQL database requires some work :

1. Create a service account as a Cloud SQL > Cloud SQL Client (only one is necessary) and save the json key file securely in your [GCP Admin](https://console.cloud.google.com/iam-admin/serviceaccounts/?authuser=0)

2. Create a MySQL User for your Database
```
$ gcloud sql connect <CLOUD SQL INSTANCE>
Whitelisting your IP for incoming connection for 5 minutes...
Connecting to database with SQL user [root].Enter password:<Your Database Password>
mysql> GRANT ALL ON db_name.* TO 'db_user'@'%' identified by 'db_secret';
mysql> FLUSH PRIVILEGES;
```

Note : I prefer doing it in mysql to fine tuned my privileges which is not available from the GCP Console

3. Create kubernetes secrets for your pods :

cloudsql-instance-credentials : Service Account from private json key file
```
kubectl create secret generic cloudsql-instance-credentials --from-file=credentials.json=[JSON KEY FILE]
```

cloudsql-db-credentials : MySql User from literal

```
kubectl create secret generic myapp-db-credentials --from-literal=username=[USERNAME] --from-literal=password=[PASSWORD]
```

4. Update your configuration pod

In the pod accessing the DB, add the following container :
```
- name: cloudsql-proxy
  image: gcr.io/cloudsql-docker/gce-proxy:1.11
  command: ["/cloud_sql_proxy",
            "-instances=[PROJECT:REGION:INSTANCE]=tcp:3306",
            "-credential_file=/secrets/cloudsql/credentials.json"]
  volumeMounts:
    - name: cloudsql-instance-credentials
    mountPath: /secrets/cloudsql
    readOnly: true
```

Change the [PROJECT:REGION:INSTANCE] with your own configuration. To find it in your Google Cloud Console, go to [SQL Instance](https://console.cloud.google.com/sql/instances/) and search for a "Remote connection" panel

Then, add the volumes section to your pod configuration :
```
volumes:
  - name: cloudsql-instance-credentials
    secret:
    secretName: cloudsql-instance-credentials
```

#### SOURCES : 
 - [Connecting from Kubernetes Engine](https://cloud.google.com/sql/docs/mysql/connect-kubernetes-engine?authuser=0)
 - [WordPress Deployment Sample](https://github.com/GoogleCloudPlatform/kubernetes-engine-samples/blob/master/cloudsql/mysql_wordpress_deployment.yaml)


### Setup Redis

We will using [stable/redis](https://github.com/helm/charts/tree/master/stable/redis) helm package

Easy peasy :
```
helm install stable/redis
```

The result :
```
** Please be patient while the chart is being deployed **
Redis can be accessed via port 6379 on the following DNS names from within your cluster:

vehement-unicorn-redis-master.default.svc.cluster.local for read/write operations
vehement-unicorn-redis-slave.default.svc.cluster.local for read-only operations


To get your password run:

    export REDIS_PASSWORD=$(kubectl get secret --namespace default vehement-unicorn-redis -o jsonpath="{.data.redis-password}" | base64 --decode)

To connect to your Redis server:

1. Run a Redis pod that you can use as a client:

   kubectl run --namespace default vehement-unicorn-redis-client --rm --tty -i \
    --env REDIS_PASSWORD=$REDIS_PASSWORD \
   --image docker.io/bitnami/redis:4.0.10-debian-9 -- bash

2. Connect using the Redis CLI:
   redis-cli -h vehement-unicorn-redis-master -a $REDIS_PASSWORD
   redis-cli -h vehement-unicorn-redis-slave -a $REDIS_PASSWORD

To connect to your database from outside the cluster execute the following commands:

    export POD_NAME=$(kubectl get pods --namespace default -l "app=redis" -o jsonpath="{.items[0].metadata.name}")
    kubectl port-forward --namespace default $POD_NAME 6379:6379
    redis-cli -h 127.0.0.1 -p 6379 -a $REDIS_PASSWORD
```

As explained, to retrieve your password :
```
kubectl get secret --namespace default vehement-unicorn-redis -o jsonpath="{.data.redis-password}" | base64 --decode
```

Modify your Laravel .env accordingly:
```
REDIS_HOST=vehement-unicorn-redis-master
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

#### Forcing SSL

With GKE LoadBalancer, we just need to add an annotation in your ingress:
```
nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
```

Here the list of available Google [nginx ingress annotations](https://github.com/kubernetes/ingress-nginx/blob/master/docs/user-guide/nginx-configuration/annotations.md)

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

## Existing Cluster

If you already have followed the "new cluster" part and you want to add a new app, you just need to follow the following points :

- Create a MySQL User for your Database
```
$ gcloud sql connect <CLOUD SQL INSTANCE>
Whitelisting your IP for incoming connection for 5 minutes...
Connecting to database with SQL user [root].Enter password:<Your Database Password>
mysql> GRANT ALL ON db_name.* TO 'db_user'@'%' identified by 'db_secret';
mysql> FLUSH PRIVILEGES;
```
- Create kubernetes secrets for your pods
```
kubectl create secret generic [APP DB CREDENTIALS] --from-literal=username=[USERNAME] --from-literal=password=[PASSWORD]
```
- Create a public ip Address
- Update the configuration yaml : app name, public ip address name, sql credentials...

## Public IP Address

How to setup a global static ip ?

Simple as:
```
gcloud compute addresses create [NAME] --global
```

Get your list of IP Addresses
```
gcloud compute addresses list
```

#### GKE LoadBalancer
If you use the Google Cloud LoadBalancer, add the following annotations in your ingress configurations:
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    kubernetes.io/ingress.global-static-ip-name: "[GLOBAL IP NAME]"
...
```

# Notes
 - GCP : Google Cloud Platform aka the Online Web Site interface
 - GKE : Google Kubernetes Engine, you can access it through GCP or kubectl with gcloud configured in your terminal