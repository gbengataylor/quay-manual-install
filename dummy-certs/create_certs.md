```
openssl genrsa -out rootCA.key 2048
# rootCA use higher level domain
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 1024 -out rootCA.pem


# Generate Quay Certs
openssl genrsa -out quay.key 2048
# For CommonName, use wildcard (for e.g. *.apps.gbengaocp43.redhatgov.io) or entire route if less than 64 bytes (for e.g., quay-enterprise-quay-quay-enterprise.apps.gbengaocp43.redhatgov.io)
openssl req -new -key quay.key -out quay.csr
# Sign the certificatie with the rootCA
openssl x509 -req -in quay.csr -CA rootCA.pem \
       -CAkey rootCA.key -CAcreateserial -out quay.crt -days 500 -sha256

#concat rootCA to Quay Cert
cat rootCA.pem >> quay.crt

# Generate Clair Certs
openssl genrsa -out clair.key 2048
# For CommonName, use quay-enterprise-clair.quay-enterprise.svc
openssl req -new -key clair.key -out clair.csr
# Sign the certificatie with the rootCA
openssl x509 -req -in clair.csr -CA rootCA.pem \
       -CAkey rootCA.key -CAcreateserial -out clair.crt -days 500 -sha256

#concat rootCA to Clair Cert
cat rootCA.pem >> clair.crt
```