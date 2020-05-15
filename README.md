**NOTE: this only works for 4.3 for now**

oc new-project quay-enterprise

Create the secret for the Red Hat Quay configuration and app
```
oc create -f quay-enterprise-config-secret.yaml
```

create the pull secret using this link https://access.redhat.com/solutions/3533201

Create the database. Note, you may need to change passwords in the files first

```
#this storage class uses ebs as default. update accordingly
#persistent storage
oc create -f quay-storageclass.yaml
oc create -f db-pvc.yaml
oc create -f postgres-deployment.yaml
oc create -f postgres-service.yaml

#run this if testing with ephermal storage
oc create -f postgres-deployment-ephemeral.yaml
oc create -f postgres-service.yaml
```

#execute this on your postgress
#change your postgress pod name TODO: determine nam

```
oc exec -it quay-enterprise-quay-postgresql-59557c97c-zjg4s  -n quay-enterprise -- /bin/bash -c 'echo "CREATE EXTENSION IF NOT EXISTS pg_trgm" | /opt/rh/rh-postgresql10/root/usr/bin/psql -d quay'
```

Create a serviceaccount for the database
```
oc create serviceaccount postgres -n quay-enterprise

oc adm policy add-scc-to-user anyuid -z system:serviceaccount:quay-enterprise:quay-enterprise-quay-postgresql
```

Create the role and binding
```
oc create -f quay-servicetoken-role-k8s1-6.yaml
oc create -f quay-servicetoken-role-binding-k8s1-6.yaml
```

add privilege
```
oc adm policy add-scc-to-user anyuid \
     system:serviceaccount:quay-enterprise:default
```

create redis
```
 oc create -f quay-enterprise-redis.yaml
 ```


Set Quay route and service
```
oc create -f quay-enterprise-service-clusterip.yaml
oc create -f quay-enterprise-app-route.yaml
```
 skip instructions to setup quay tool
 ```
#oc create -f quay-enterprise-config.yaml
#oc create -f quay-enterprise-config-service-clusterip.yaml
#oc create -f quay-enterprise-config-route.yaml
```

 instead use the config.yaml in the repo. modify accordingly. may need to update route and storage option

```
oc delete secret quay-enterprise-config-secret
oc create secret generic  quay-enterprise-config-secret --from-file="config.yaml=config.yaml"
```

also need to manually add superuser to the  postgres table..TODO

Start Quay
 ```
oc create -f quay-enterprise-app-rc.yaml
```


TODO: Determine how to set the superuser so that quay doesn't fail on startup
      See what db updates the quay tool performs