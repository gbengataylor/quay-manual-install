**NOTE: this only works for 4.3 for now. The API for Deployments changed in 4.4 (k8s 1.17) so deployments will fail **

TODO: Organize in different folders

Create Quay Project.
```
oc new-project quay-enterprise
oc project quay-enterprise
```

Create the secret for the Red Hat Quay configuration and app
```
oc create -f quay-enterprise-config-secret.yaml -n quay-enterprise
```

create the pull secret using this link https://access.redhat.com/solutions/3533201


Create the database. Note, you may need to change passwords in the files first

```
#this storage class uses ebs as default. update accordingly to your cloud provider. Also included is ephermeral storage for quick testing.

oc create -f postgres/ 

#persistent storage
oc create -f quay-storageclass.yaml
oc create -f postgres/persistent

#run this if testing with ephermal storage
oc create -f postgres/ephemeral
```

#execute this on your postgress

#change your postgress pod name AFTER the posgresql pod is running
```
postgres_pod=$(oc get pods -n quay-enterprise  -lapp=quay-enterprise-quay-postgresql | grep quay-enterprise-quay-postgresql | awk '{ print $1}')
oc exec -it $postgres_pod  -n quay-enterprise -- /bin/bash -c 'echo "CREATE EXTENSION IF NOT EXISTS pg_trgm" | /opt/rh/rh-postgresql10/root/usr/bin/psql -d quay'
```

Create a serviceaccount for the database
```
oc create serviceaccount postgres -n quay-enterprise
oc adm policy add-scc-to-user anyuid -z system:serviceaccount:quay-enterprise:quay-enterprise-quay-postgresql
```

Create the role and binding
```
oc create -f service-token/
```

add privilege
```
oc adm policy add-scc-to-user anyuid \
     system:serviceaccount:quay-enterprise:default
```

create redis
```
 oc create -f redis/
```

Set Quay route and service
```
oc create -f quay-enterprise-service-clusterip.yaml
oc create -f quay-enterprise-app-route.yaml
```
 skip instructions to setup quay tool
 ```
#oc create -f config-tool/
```

instead use the config-copied-clair.yaml in the repo. modify accordingly. may need to update route and storage option.
Also, supply your appropriate certs. For testing you can use dummy certs but best to follow instructions here to generate the certs:
https://access.redhat.com/documentation/en-us/red_hat_quay/3.3/html-single/manage_red_hat_quay/index#using-ssl-to-protect-quay
Finally this example uses localStorage, which isn't supported. In production env, update to use a supported storage backend for the registry
```
# probably didn't need to create it the first time but leaving as-is
oc delete secret quay-enterprise-config-secret
oc create secret generic  quay-enterprise-config-secret --from-file="config.yaml=config-copied-clair.yaml" \
                                    --from-file=ssl.key=dummy-certs/ca/device.key \
                                    --from-file=ssl.cert=dummy-certs/ca/device.crt
# if you generate extra certs use this as an example. still use the appropriate certs
#oc create secret generic  quay-enterprise-config-secret --from-file="config.yaml=config-copied-clair.yaml" \
                                    --from-file=ssl.key=dummy-certs/ssl.key \
                                    --from-file=ssl.cert=dummy-certs/ssl.cert \
                                    --from-file=extra_ca_certs_quay.crt=dummy-certs/extra_ca_certs_quay.crt
```



Start Quay
 ```
oc create -f quay-enterprise-app-rc.yaml
```

Quay will start crashing becuase we need to insert superuser. Need to do this AFTER initially Quay deploys since it creates the Schema for the Quay app in postgres
```
postgres_pod=$(oc get pods -n quay-enterprise  -lapp=quay-enterprise-quay-postgresql | grep quay-enterprise-quay-postgresql | awk '{ print $1}')
#note: currently uses hash for password..for demo, hardcoding for now
INSERT_SQL='INSERT INTO "user" ("uuid", "username", "email", "verified", "organization", "robot", "invoice_email", "invalid_login_attempts", "last_invalid_login", "removed_tag_expiration_s", "enabled" , "creation_date", "password_hash") VALUES ('c2e14e33-a155-4d38-9cc5-d44b00cbe360', 'quay', 'changeme@example.com', true, false, false, false, 0, current_timestamp, 1209600, true,current_timestamp, '$2a$12$u0HMSBz3slLA8jyf4Gi/6.GDbZAM2u8OZDfC6i/oCujqFHVS/CP3W');'

oc exec -it $postgres_pod -n quay-enterprise  -- /bin/bash -c 'echo $INSERT_SQL | psql -d quay'

#verify it worked 
SELECT_SQL='select id, uuid, username, email, verified, organization, robot, invoice_email, invalid_login_attempts, last_invalid_login, removed_tag_expiration_s, enabled from "user";'
oc exec -it $postgres_pod -n quay-enterprise  -- /bin/bash -c 'echo $SELECT_SQL | psql -d quay'

#you might need to log into pod and execute and view command
oc rsh $postgres_pod

# if you changed your database name and/or user, change appropriately
sh-4.2$ psql -d quay -U quay

## execute $INSERT_SQL in pssql shell


quay==> \q
sh-4.2$ exit
```

Give ```quay-enterprise-app``` a few seconds to boot up successfully. Once it does, you should be able to login as quay/password

TODO: add Clair and test instructions

Create the Clair Database. You may need to change the database credentials if they were modified. Also, as with the quay database, you may need to modify for the appropriate storage type for the cloud provider. For testing, use the ephemeral option if you don't have a persistent storage on your cluster

```
# service 
oc create -f clair/postgres

#persisent
oc create -f clair/postgres/persistent

#ephemeral
oc create -f clair/postgres/ephemeral
```

The config.yaml in this repo should already have the appropriate clair settings. You will need to update your clair-config.yaml with the appropriate quay route

Create the clair config secret, service, and deployment
```
oc create secret generic clair-scanner-config-secret \
   --from-file=config.yaml=clair/clair-config.yaml \
   --from-file=security_scanner.pem=dummy-certs/security_scanner.pem \
   --from-file=kid=dummy-certs/security_scanner.id
oc create -f clair/clair-service.yaml
oc create -f clair/clair-deployment.yaml
```

restart quay
```
oc delete pod -lquay-enterprise-component=app
```
Wait till the quay app pod restarts, login and attempt to push a new image to a repo. If successful, the clair scan should get Queued and at some point the scan should happen

# APPENDIX

```
#in quay-enterprise-quay-postgresql

psql -d quay -U quay

#in psql prompt
SELECT table_name FROM information_schema.tables WHERE table_schema='public';
select table_name, column_name, table_schema, data_type, is_nullable from information_schema.columns where table_name='user';


SELECT table_name, table_schema FROM information_schema.tables; 

select id, uuid, username, email, verified, organization, robot, invoice_email, invalid_login_attempts, last_invalid_login, removed_tag_expiration_s, enabled from public.user;

```
