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

#change your postgress pod name TODO: determine nam

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

instead use the config.yaml in the repo. modify accordingly. may need to update route and storage option
Also, supply your appropriate certs. 
```
oc delete secret quay-enterprise-config-secret
oc create secret generic  quay-enterprise-config-secret --from-file="config.yaml=config.yaml" \
                                    --from-file=ssl.key=dummy-certs/ssl.key \
                                    --from-file=ssl.cert=dummy-certs/ssl.cert
```



Start Quay
 ```
oc create -f quay-enterprise-app-rc.yaml
```


Quay will start crashing, need to insert superuser. Need to do this AFTER Quay deploys since it creates the Schema
```
postgres_pod=$(oc get pods -n quay-enterprise  -lapp=quay-enterprise-quay-postgresql | grep quay-enterprise-quay-postgresql | awk '{ print $1}')
#note: currently uses hash for password..for demo, hardcoding for now
INSERT_SQL='INSERT INTO "user" ("uuid", "username", "email", "verified", "organization", "robot", "invoice_email", "invalid_login_attempts", "last_invalid_login", "removed_tag_expiration_s", "enabled" , "creation_date", "password_hash") VALUES ('f85a601e-90c2-472e-8f11-c5c9d2ffa7bc', 'quay', 'changeme@example.com', false, false, false, false, 0, current_timestamp, 1209600, true,current_timestamp, 'a$12$aijsRWiXXYj5l83WezZ1juG2pR37DSgOvHtC9PpEjs1FXMtJ1O');'

oc exec -it $postgres_pod -n quay-enterprise  -- /bin/bash -c 'echo $INSERT_SQL | psql -d quay'

#verify it worked 
SELECT_SQL='select id, uuid, username, email, verified, organization, robot, invoice_email, invalid_login_attempts, last_invalid_login, removed_tag_expiration_s, enabled from "user";'
oc exec -it $postgres_pod -n quay-enterprise  -- /bin/bash -c 'echo $SELECT_SQL | psql -d quay'

#you might need to log into pod and execute and view command
oc rsh $postgres_pod

sh-4.2$ psql -d quay -U quay

## execute INSERT STATEMENT in pssql shell

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
