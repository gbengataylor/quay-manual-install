oc project $quay-project

Create the secret for the Red Hat Quay configuration and app
```oc create -f quay-enterprise-config-secret.yaml
```

create the pull secret using this link https://access.redhat.com/solutions/3533201

Create the database. Note, you may need to change passwords in the files first

```
oc create -f quay-storageclass.yaml
oc create -f db-pvc.yaml
oc create -f postgres-deployment.yaml
oc create -f postgres-service.yaml
```