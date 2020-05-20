This repo has instructions on how to manually install Quay on OpenShift without the use of the Operator and also without the Quay Setup Tool

**NOTE: this only works for 4.3 and below for now as it uses the old API Spec for Deployments. That api was completely deprecated and removed in 4.4 (k8s 1.17) so deployments will fail if using the Deployment resources in this repo. Need to update the Deployment Resources with New API spec **

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

#run this if testing the database with ephermal storage
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

instead use the ```config-copied.yaml``` in the repo. modify accordingly. You need to update route and storage option (if not using LocalStorage).
```
# you may also need to change the storage config
DISTRIBUTED_STORAGE_CONFIG:
  default:
  - LocalStorage
  - storage_path: /datastorage/registry
DISTRIBUTED_STORAGE_DEFAULT_LOCATIONS: []
DISTRIBUTED_STORAGE_PREFERENCE:
- default
...
...
# replace this with quay route
SERVER_HOSTNAME: quay-enterprise-quay-quay-enterprise.apps.gbengaocp43.redhatgov.io
```

Also, supply your appropriate certs. For testing you can use dummy certs but best to follow instructions here to generate the certs:
https://access.redhat.com/documentation/en-us/red_hat_quay/3.3/html-single/manage_red_hat_quay/index#using-ssl-to-protect-quay
Finally this example uses localStorage, which isn't supported. In production env, update to use a supported storage backend for the registry
```
# probably didn't need to create it the first time but leaving as-is
oc delete secret quay-enterprise-config-secret
# extra_ca_certs_quay.crt and ssl.cert should be same value
oc create secret generic  quay-enterprise-config-secret --from-file="config.yaml=config-copied.yaml" \
                                    --from-file=ssl.key=dummy-certs/ca/device.key \
                                    --from-file=ssl.cert=dummy-certs/ca/device.crt \
                                    --from-file=extra_ca_certs_quay.crt=dummy-certs/ca/device.crt
```

 

Start Quay
 ```
oc create -f quay-enterprise-app-rc.yaml
```

Quay will start crashing becuase we need to insert superuser. Need to do this AFTER the first time Quay deploys as it creates the Schema for the Quay app in the database
```
postgres_pod=$(oc get pods -n quay-enterprise  -lapp=quay-enterprise-quay-postgresql | grep quay-enterprise-quay-postgresql | awk '{ print $1}')
# note: currently uses hash for password..for demo, hardcoding for now
INSERT_SQL='INSERT INTO "user" ("uuid", "username", "email", "verified", "organization", "robot", "invoice_email", "invalid_login_attempts", "last_invalid_login", "removed_tag_expiration_s", "enabled" , "creation_date", "password_hash") VALUES ('c2e14e33-a155-4d38-9cc5-d44b00cbe360', 'quay', 'changeme@example.com', true, false, false, false, 0, current_timestamp, 1209600, true,current_timestamp, '$2a$12$u0HMSBz3slLA8jyf4Gi/6.GDbZAM2u8OZDfC6i/oCujqFHVS/CP3W');'

oc exec -it $postgres_pod -n quay-enterprise  -- /bin/bash -c 'echo $INSERT_SQL | psql -d quay'

# verify it worked 
SELECT_SQL='select id, uuid, username, email, verified, organization, robot, invoice_email, invalid_login_attempts, last_invalid_login, removed_tag_expiration_s, enabled from "user";'
oc exec -it $postgres_pod -n quay-enterprise  -- /bin/bash -c 'echo $SELECT_SQL | psql -d quay'

# you might need to log into pod and execute and view command
oc rsh $postgres_pod

# if you changed your database name and/or user, change appropriately
sh-4.2$ psql -d quay -U quay

## execute $INSERT_SQL in pssql shell


quay==> \q
sh-4.2$ exit
```

Give ```quay-enterprise-app``` a few seconds to boot up successfully. Once it does, you should be able to login as quay/password


Create the Clair Database. You may need to change the database credentials if they were modified. Also, as with the quay database, you may need to modify for the appropriate storage type for the cloud provider. For testing, use the ephemeral option if you don't have a persistent storage on your cluster

```
# service 
oc create -f clair/postgres

#persisent
oc create -f clair/postgres/persistent

#ephemeral
oc create -f clair/postgres/ephemeral
```

You will need to update your ```clair-config.yaml``` with the appropriate quay route
```
    http:
      # modify with quay app endpoint
      endpoint: HTTPS://quay-enterprise-quay-quay-enterprise.apps.gbengaocp43.redhatgov.io/secscan/notify
   ...

     key_server:
        type: keyregistry
        options:
          # modify with quay app endpoint
          registry: https://quay-enterprise-quay-quay-enterprise.apps.gbengaocp43.redhatgov.io/keys/
```

Create the clair config secret, service, and deployment

You will need to create another certificate for clair using the common name: quay-enterprise-clair.quay-enterprise.svc. For testing use the dummy certs

The ```security_scanner.pem``` file and key can be generated from the clair config tool. You can use the ones in this repo
```
# using the same certs for quay here (tls.crt, tls.key). update if certs where generated
# There are two security_scanner files in this repo. Execute only one of the commands

#using security_scanner
oc create secret generic clair-scanner-config-secret \
   --from-file=config.yaml=clair/clair-config.yaml \
   --from-file=security_scanner.pem=dummy-certs/security_scanner.pem \
   --from-file=kid=dummy-certs/security_scanner.id \
   --from-file=tls.crt=dummy-certs/clair-ca/device.crt \
   --from-file=tls.key=dummy-certs/clair-ca/device.key

#using security_scanner-1
#oc create secret generic clair-scanner-config-secret \
   --from-file=config.yaml=clair/clair-config-1.yaml \
   --from-file=security_scanner.pem=dummy-certs/security_scanner-1.pem \
   --from-file=kid=dummy-certs/security_scanner-1.id \
   --from-file=tls.crt=dummy-certs/clair-ca/device.crt \
   --from-file=tls.key=dummy-certs/clair-ca/device.key

# Create the Clair Service and Deployment
oc create -f clair/clair-service.yaml
oc create -f clair/clair-deployment.yaml
```

Verify clair is running in the logs
```
time="2020-05-20T14:49:22Z" level=info msg="Starting reverse proxy (Listening on ':6060')" 
time="2020-05-20T14:49:22Z" level=info msg="Starting forward proxy (Listening on ':6063')"
...
...
...
{"Event":"Start fetching vulnerabilities","Level":"info","Location":"rhel.go:92","Time":"2020-05-20 14:49:22.880502","package":"RHEL"}
```

Update ```config-copied-clair.yaml``` as needed. 

```
# you may also need to change the storage config
DISTRIBUTED_STORAGE_CONFIG:
  default:
  - LocalStorage
  - storage_path: /datastorage/registry
DISTRIBUTED_STORAGE_DEFAULT_LOCATIONS: []
DISTRIBUTED_STORAGE_PREFERENCE:
- default
...
...
...
# replace this with quay route
SERVER_HOSTNAME: quay-enterprise-quay-quay-enterprise.apps.gbengaocp43.redhatgov.io
```

Regenerate config secret with new config file with clair and add the clair certs to the secret. Quay needs this cert to be able to invoke the clair scan
(Also could have started with ```config-copied-clair.yaml``` to avoid updates)
```
oc delete secret quay-enterprise-config-secret
# extra_ca_certs_quay.crt and ssl.cert should be same value
# extra_ca_certs_clair.crt should point to same cert as clair
oc create secret generic  quay-enterprise-config-secret --from-file="config.yaml=config-copied-clair.yaml" \
                                    --from-file=ssl.key=dummy-certs/ca/device.key \
                                    --from-file=ssl.cert=dummy-certs/ca/device.crt \
                                    --from-file=extra_ca_certs_quay.crt=dummy-certs/ca/device.crt \
                                    --from-file=extra_ca_certs_clair.crt=dummy-certs/clair-ca/device.crt 
```

Redeploy the quay pod so latest changes are applied
```
#oc delete pod -lquay-enterprise-component=app
```
Wait till few minutes after the Quay pod restarts, login and attempt to push a new image to a repo. If successful, the security scan column should be set to Queued and at some point the scan should happen. If you check the Quay logs, there should be no ```secsan``` errors and the Clair pod log should look like

```
{"Event":"Handled HTTP request","Level":"info","Location":"router.go:57","Time":"2020-05-20 14:15:01.599182","elapsed time":20569796,"method":"POST","remote addr":"[::1]:55154","request uri":"/v1/layers","status":"422"}
{"Event":"Handled HTTP request","Level":"info","Location":"router.go:57","Time":"2020-05-20 14:15:05.870369","elapsed time":4240433278,"method":"POST","remote addr":"[::1]:55160","request uri":"/v1/layers","status":"201"}
....
```


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
