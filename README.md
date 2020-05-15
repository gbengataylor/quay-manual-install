**NOTE: this only works for 4.3 for now**

TODO: Organize in different folders

Create Quay Project.
```
oc new-project quay-enterprise
```

Create the secret for the Red Hat Quay configuration and app
```
oc create -f quay-enterprise-config-secret.yaml
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
oc exec -it quay-enterprise-quay-postgresql-59557c97c-zjg4s  -n quay-enterprise -- /bin/bash -c 'echo "CREATE EXTENSION IF NOT EXISTS pg_trgm" | /opt/rh/rh-postgresql10/root/usr/bin/psql -d quay'
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

```
oc delete secret quay-enterprise-config-secret
oc create secret generic  quay-enterprise-config-secret --from-file="config.yaml=config.yaml"
```

also need to manually add superuser to the  postgres table..TODO
```
#TODO
INSERT INTO "user" ("uuid", "username", "email", "verified", "organization", "robot", "invoice_email", "invalid_login_attempts", "last_invalid_login", "removed_tag_expiration_s", "enabled", "maximum_queued_builds_count", "creation_date") VALUES ('f85a601e-90c2-472e-8f11-c5c9d2ffa7bc', "quay", "changeme@example.com", false, false, false, false, %s, %s, %s, %s, %s, %s) RETURNING "user"."id"', [u'f85a601e-90c2-472e-8f11-c5c9d2ffa7bc', u'quay', u'changeme@example.com', False, False, False, False, 0, datetime.datetime(2020, 5, 15, 11, 25, 37, 548249), 1209600, True, None, datetime.datetime(2020, 5, 15, 11, 25, 37, 548245)])
```

Start Quay
 ```
oc create -f quay-enterprise-app-rc.yaml
```


TODO: Determine how to set the superuser so that quay doesn't fail on startup
     
      See what db updates the quay tool performs

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

INSERT INTO "user" ("id", "uuid", "username", "email", "verified", "organization", "robot", "invoice_email", "invalid_login_attempts", "last_invalid_login", "removed_tag_expiration_s", "enabled" , "creation_date") VALUES (1, 'f85a601e-90c2-472e-8f11-c5c9d2ffa7bc', "quay", "changeme@example.com", false, false, false, false, 0, current_timestamp, true,current_timestamp) ;
```
