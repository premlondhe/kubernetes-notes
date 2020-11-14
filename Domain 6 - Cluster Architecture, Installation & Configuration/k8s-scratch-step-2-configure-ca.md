##### 0. Create base directory were all the certificates and keys will be stored

```sh
mkdir /root/certificates
cd /root/certificates
```

##### 1. Creating a private key for Certificate Authority
```sh
openssl genrsa -out ca.key 2048
```
##### 2. Creating CSR
```sh
openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr
```
Note: If you are in ubuntu environment and get below error than, just comment out two lines starting with **RANDFILE** from ```/etc/ssl/openssl.cnf``` file
```sh
Can't load /root/.rnd into RNG
139991619654080:error:2406F079:random number generator:RAND_load_file:Cannot open file:../crypto/rand/randfile.c:88:Filename=/root/.rnd
```

##### 3. Self-Sign the CSR
```sh
openssl x509 -req -in ca.csr -signkey ca.key -CAcreateserial  -out ca.crt -days 1000
```
