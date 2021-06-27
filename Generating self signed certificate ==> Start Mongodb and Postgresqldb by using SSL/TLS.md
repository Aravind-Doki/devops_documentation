## Generating self signed certificate ==> Start Mongodb and Postgresqldb by using SSL/TLS
We will be using OpenSSL to create own private certificate authority.
The process for creating your own certificate authority is
- Create a private key
- Self-sign
- Install root CA on your various workstations

## Create a Private key/Root Key:
- Attention: this is the key used to sign the certificate requests, anyone holding this can sign certificates on your behalf. So keep it in a safe place!
```sh
openssl genrsa -des3 -out rootCA.key 4096
```
- If you want a non password protected key just remove the -des3 option

## Self-sign:
```sh
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 1024 -out rootCA.pem
```
- Here we used our root key to create the root certificate that needs to be distributed in all the computers that have to trust us.
- This will start an interactive script which will ask you for various bits of information. Fill it out as you see fit.

Country Name (2 letter code) [AU]:INState or Province Name (full name) [Some-State]:TSLocality Name (eg, city) []:HYDOrganization Name (eg, company) [Internet Widgits Pty Ltd]:ILOrganizational Unit Name (eg, section) []:ITCommon Name (eg, YOUR name) []: IP addressEmail Address []:devopsil@company.com

## Install root CA on your various workstations:

## Create A Certificate (Done Once Per Device):

- This procedure needs to be followed for each server/appliance that needs a trusted certificate from our CA.
```sh
openssl genrsa -out company.key 2048
```
- Once the key is created, you’ll generate the certificate signing request.
```sh
openssl req -new -key company.key -out company.csr
```
- This will start an interactive script which will ask you for various bits of information. Fill it out as you see fit.

Country Name (2 letter code) [AU]:INState or Province Name (full name) [Some-State]:TSLocality Name (eg, city) []:HYDOrganization Name (eg, company) [Internet Widgits Pty Ltd]:ILOrganizational Unit Name (eg, section) []:ITCommon Name (eg, YOUR name) []:IP addressEmail Address []:devopsil@company.com

- Whatever you see in the address field in your browser when you go to your device must be what you put under common name, even if it’s an IP address. Yes, even an IP (IPv4 or IPv6) address works under common name. If it doesn’t match, even a properly signed certificate will not validate correctly and you’ll get the “cannot verify authenticity” error. Once that’s done, you’ll sign the CSR, which requires the CA root key.

## Generate the certificate using the company csr and key along with the CA Root key:
```sh
openssl x509 -req -in company.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out company.crt -days 500 -sha256
```
- This creates a signed certificate called device.crt which is valid for 500 days (you can adjust the number of days of course, although it doesn’t make sense to have a certificate that lasts longer than the root certificate).

## Create .pem file:
```sh
cat company.key company.crt > company.pem
```
=====================================================================

## Enabling TLS on MongoDB server:

- Open /etc/mongod.conf with code editor and make sure it contains the following lines:
```sh
sudo vi /etc/mongod.conf
```
- under net:
port:27017
bindIp: 127.0.0.1,server_public_IP
port:27017bindIp: 127.0.0.1,server_public_IPtls:mode: requireTLScertificateKeyFile: <path>/company.pemCAFile: <path>/rootCA.pemallowInvalidCertificates: trueallowInvalidHostnames: true

- Under security add below line:

security:
authorization: enabled

- Now you are ready to save the configuration file and restart mongodb by using below command:
```sh
sudo systemctl restart mongod
```
## Using TLS with the mongo shell:
```sh
mongo --tls --tlscertificateKeyFile <path>/mongodb.pem --tlsCAFile: <path>/rootCA.pem --tlsAllowInvalidHostnames --tlsAllowInvalidCertificates --host <IPaddress>
```
- --tls enables TLS channel encryption.

- --tlscertificateKeyFile is the path to the client certificate — which needs to be signed by the server certificate.

- --tlsCAFile is the path to the root certificate of the Certification Authority (CA) that signed the server certificate. From MongoDB, this parameter is optional, and if not specified, the client will check the certificate against the system CA store.

- --host (optional) verifies that the hostname of the server matches the one in the certificate it presents.

=====================================================================

## Enabling TLS on Postgresdb server:

- Edit PostgreSQL configuration by using code editor
```sh
sudo nano /etc/postgresql/10/main/postgresql.conf
```
- Add the below parameters in the configuration file.
```sh
listen_addresses = 'localhost,SERVER_PUBLIC_IP' or '*'
ssl_cert_file = '<path>/company.pem'
ssl_key_file = '<path>/company.csr'
ssl_ca_file = '<path>/rootCA.pem'
```
- Now you are ready to save the configuration file and restart Postgresdb by using below command:
```sh
sudo systemctl restart postgresql
or
/etc/init.d/postgresql restart
```
- Now edit pg_hba.conf to force SSL for our remote user:
```sh
nano /etc/postgresql/10/main/pg_hba.conf
```
- Locate the IPv4 connections and add a new one:
```sh
host    all             all             127.0.0.1/32            md5                # <-- existing line
hostssl all             user_name     [CLIENT_IP_ADDRESS]/32  md5 clientcert=1   # <-- new line
```
- Now you are ready to save the configuration file and restart Postgresdb by using below command:
```sh
sudo systemctl restart postgresql
or
/etc/init.d/postgresql restart
```
- Check the logs
```sh
$ tail /var/log/postgresql/postgresql-10-main.log 
```
- Using TLS with the psql:
```sh
psql *ssl_cert_file=<path>/company.pem ssl_key_file=<path>/company.csr ssl_ca_file=<path>/rootCA.pem sslmode=verify-full* -U user_name -w -h <IPaddress>
```
