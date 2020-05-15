**NOTE: this only works for 4.3 for now**

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
#this storage class uses ebs as default. update accordingly

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

instead use the config.yaml in the repo. modify accordingly. may need to update route and storage option.
Also, supply your appropriate certs. For testing you can use dummy certs but best to follow instructions here to generate the certs:
https://access.redhat.com/documentation/en-us/red_hat_quay/3.3/html-single/manage_red_hat_quay/index#using-ssl-to-protect-quay
```
# probably didn't need to create it the first time but leaving as-is
oc delete secret quay-enterprise-config-secret
oc create secret generic  quay-enterprise-config-secret --from-file="config.yaml=config-copied.yaml" \
                                    --from-file=ssl.key=dummy-certs/ca/device.key \
                                    --from-file=ssl.cert=dummy-certs/ca/device.crt
# if you generate extra certs use this as an example. still use the appropriate certs
#oc create secret generic  quay-enterprise-config-secret --from-file="config.yaml=config-copied.yaml" \
                                    --from-file=ssl.key=dummy-certs/ssl.key \
                                    --from-file=ssl.cert=dummy-certs/ssl.cert \
                                    --from-file=extra_ca_certs_quay.crt=dummy-certs/extra_ca_certs_quay.crt
```



Start Quay
 ```
oc create -f quay-enterprise-app-rc.yaml
```


Quay will start crashing, need to insert superuser. Need to do this AFTER Quay deploys since it creates the Schema
```
postgres_pod=$(oc get pods -n quay-enterprise  -lapp=quay-enterprise-quay-postgresql | grep quay-enterprise-quay-postgresql | awk '{ print $1}')
#note: currently uses hash for password..for demo, hardcoding for now
INSERT_SQL='INSERT INTO "user" ("uuid", "username", "email", "verified", "organization", "robot", "invoice_email", "invalid_login_attempts", "last_invalid_login", "removed_tag_expiration_s", "enabled" , "creation_date", "password_hash") VALUES ('c2e14e33-a155-4d38-9cc5-d44b00cbe360', 'quay', 'changeme@example.com', true, false, false, false, 0, current_timestamp, 1209600, true,current_timestamp, '$2a$12$u0HMSBz3slLA8jyf4Gi/6.GDbZAM2u8OZDfC6i/oCujqFHVS/CP3W');'

#c2e14e33-a155-4d38-9cc5-d44b00cbe360
#$2a$12$u0HMSBz3slLA8jyf4Gi/6.GDbZAM2u8OZDfC6i/oCujqFHVS/CP3W

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



TODO: add Clair and test instructions

Creat the Clair Database TODO: ephmeral also
```
oc create -f clair/postgres-clair-storage.yaml
oc create -f clair/postgres-clair-deployment.yaml
oc create -f clair/postgres-clair-service.yaml
```

instructions to opn the quay setup ui to enable scanning, need to make sure that config.yaml reflects this

instructions to edit clair-config.yaml


Create the clair config secret and service

```
# likely need to change settings..might need to user clair-config-2.yaml
oc create secret generic clair/clair-scanner-config-secret \
   --from-file=config.yaml=/path/to/clair-config.yaml \
   --from-file=security_scanner.pem=/path/to/security_scanner.pem
oc create -f clair/clair-service.yaml
oc create -f clair/clair-deployment.yaml
```

instructions to get the clair-service endpoint, enter security scanner endpoint ...make sure that config.yaml reflects this

restart quay


#random stuff

```
#in quay-enterprise-quay-postgresql

psql -d quay -U quay

#in psql prompt
SELECT table_name FROM information_schema.tables WHERE table_schema='public';
select table_name, column_name, table_schema, data_type, is_nullable from information_schema.columns where table_name='user';


SELECT table_name, table_schema FROM information_schema.tables; 

select id, uuid, username, email, verified, organization, robot, invoice_email, invalid_login_attempts, last_invalid_login, removed_tag_expiration_s, enabled from public.user;

```
