# Setup your Docker Laravel app in Kubernetes

How to setup the app :

1. Create a secret for your Docker Private Registry (default namespace)
2. [GKE/Production ONLY]Create your public ip address + DNS
3. Create/Update a ConfigMap for your Laravel .env
4. Apply your configuration
5. Import your DB or Migrate from a container

## Docker Private Registry Secret: myteam-gitlab
We need to store the credentials of gitlab in our cluster, the secret will be labeled "myteam-gitlab" and can be used by any project.

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

## Create your public ip address + DNS

On Google, it's pretty easy
```
gcloud compute addresses create [NAME] --global
```

Get your list of IP Addresses
```
gcloud compute addresses list
```

Then go to your DNS provider and register this new ip address with your domain name.

## Laravel .env Config

Two solutions here:
  1. Create a ConfigMap yaml Configuration file
  2. Import a file

### ConfigMap

I have chosen this case to maintain only one big configuration file.

Here a talking sample of a yaml ConfigMap:
```
apiVersion: v1
data:
  .env: |-
    APP_ENV=production
    APP_KEY=base64:xxxxxxxxxxxxxxxx
    APP_DEBUG=false
    APP_URL=[URL]
    APP_LOG=errorlog

    DB_CONNECTION=mysql
    DB_HOST=127.0.0.1 # keep it that way for Cloud SQL Proxy in the same pod
    DB_PORT=3306
    DB_DATABASE=[NAME]
    DB_USERNAME=[LOGIN]
    DB_PASSWORD=[PASSWORD]

    # Add other ENV variables as needed
kind: ConfigMap
metadata:
  name: myapp-conf
  namespace: default
```

### Import a file
#### Create
We need to store our .env config (labeled myapp-env)
```
kubectl create configmap myapp-env --from-file=./.env
```

#### Update
To update a configuration, you'll have to delete it before recreating it:
```
kubectl delete configmap myapp-env
```

Then, you'll need to refresh your pods... Then shortest solution at this time is to delete linked deployments and then recreate it again:
```
kubectl delete -f myapp.conf
kubectl apply -f myapp.conf
```

Another solution is to create a new configuration map (new name) and modify your deployments accordingly like you'll do in docker stack with your configs...

#### Export to yaml
```
kubectl get cm myapp-env -o yaml
```

#### Warning
In docker, we reference the service name (instances of containers) for network communication, in kubernetes we really use the service declared. 

for kubernetes, in your .env file reference the DB_HOST with the database service :
```
DB_HOST=myapp-db-svc
```

## Apply the configuration

Modify the template accordingly then apply it:
```
kubectl apply -f myapp.conf
```

## Database

### Import your SQL File
#### Minkube : inject DB in the db-pod
```
kubectl exec -i <DB POD NAME> -- mysql -p[DB PASSWORD] [DB NAME] < <SQL FILE>
```

#### GKE

There are 2 solutions :

 1. Import the DB from the [Google Cloud Store](https://console.cloud.google.com/storage/browser)
   - Create a cloud store bucket if necessary
   - Upload your sql file in your bucket storage
   - Go back to your [SQL Instance](https://console.cloud.google.com/sql/instances/) and Import
 2. Connect to your SQL Instance with gcloud
```
gcloud sql connect <DB INSTANCE>
mysql> SOURCE <SQL FILE>
```

If you have some errors, try to clean your file and re-import it

##### Clean your SQL File

Remove all trigger definers
```
sed -i -e 's/DEFINER=[^ ]* / /' [SQL FILE]
```

SOURCE: 
I was searching for a better solution like a new parameter for my mysqldump and I discovered the mysqlpump command in this thread : [Remove DEFINER clause from MySQL Dumps](https://stackoverflow.com/questions/9446783/remove-definer-clause-from-mysql-dumps)

### Migrate

Find your pod where do you want to launch your artisan command
```
kubectl get pods
```

Launch your migrate (I assume your Laravel app is in /var/www/html/)
```
kubectl exec -it [POD NAME] -- /var/www/html/php artisan migrate
```
