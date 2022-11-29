# Creating a Certificate Authority and Client Certificates with OpenSSL

Initial source: https://medium.com/cyberark-engineering/calling-aws-services-from-your-on-premises-servers-using-iam-roles-anywhere-3e335ed648be#1582

Appendix: Creating a Certificate Authority and Client Certificates with OpenSSL
Note: This is only for testing purposes! By no means should this implementation be used for anything other than that. In case you are rolling your own private CA, take the time to investigate your needs and requirements before deciding how to implement it. THIS IS ONLY FOR TESTING PURPOSES!

Let’s create a simple certificate authority (CA) using OpenSSL, and use it to sign client certificates, which we can use to test IAM Roles Anywhere. Note that IAM Roles Anywhere requires that we use a version 3 CA certificate.

1. Generate an elliptic curve key for the certificate authority:

```
openssl ecparam -genkey -name secp384r1 -out rootCA.key
```

2. Create a file called root_request.config, which will serve as the config for the CA certificate signing request (CSR). Paste the following into it:

```
[req]
prompt             = no
string_mask        = default
distinguished_name = req_dn
[req_dn]
countryName = US
stateOrProvinceName = NY
localityName = New York
organizationName = Acme Inc.
organizationalUnitName = AC
commonName = Acme CA
```

3. Create the CSR for the CA certificate:

```
openssl req -new -key rootCA.key -out rootCA.req -nodes -config root_request.config
```

4. To create the CA certificate, we require creation of a database file and a serial file (the CA requires those for signing certs), run the following commands:

```
touch index
echo 01 > serial.txt
```

5. Create the CA certificate config file named root_certificate.config and paste the following into it:

# This is used with the 'openssl ca' command to sign a request

```
[ca]
default_ca = CA
[CA]
# Where OpenSSL stores information
dir             = .
certs           = $dir
crldir          = $dir
new_certs_dir   = $certs
database        = $dir/index
certificate     = $certs/rootcrt.pem
private_key     = $dir/rootprivkey.pem
crl             = $crldir/crl.pem   
serial          = $dir/serial.txt
RANDFILE        = $dir/.rand
# How OpenSSL will display certificate after signing
name_opt    = ca_default
cert_opt    = ca_default
# How long the CA certificate is valid for
default_days = 3650
# The message digest for self-signing the certificate
default_md = sha256
# Subjects don't have to be unique in this CA's database
unique_subject    = no
# What to do with CSR extensions
copy_extensions    = copy
# Rules on mandatory or optional DN components
policy      = simple_policy
# Extensions added while singing with the `openssl ca` command
x509_extensions = x509_ext
[simple_policy]
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = optional
domainComponent         = optional
emailAddress            = optional
name                    = optional
surname                 = optional
givenName               = optional
dnQualifier             = optional
[ x509_ext ]
# These extensions are for a CA certificate
subjectKeyIdentifier    = hash
authorityKeyIdentifier  = keyid:always
basicConstraints            = critical, CA:TRUE
keyUsage = critical, keyCertSign, cRLSign, digitalSignature
```

6. Create the CA certificate (answer yes to all prompts):

```
openssl ca -out rootCA.pem -keyfile rootCA.key -selfsign -config root_certificate.config  -in rootCA.req
```

7. Examine your CA certificate details. Note that the version is 3 (0x2):

```
openssl x509 -noout -text -in rootCA.pem
```

8. Create the client key:

```
openssl ecparam -genkey -name secp384r1 -out server.key
```

9. Create the client CSR config file named server_request.config. This is the most basic and minimal CSR configuration, just for testing purposes. Paste the following:

```
[ req ]
prompt = no
distinguished_name = dn
[ dn ]
C = US
O = Acme
CN = Acme.com
OU = Innovation
```

Note how the OU is “Innovation” to comply with our trust policy.

10. Create the client CSR:

```
openssl req -new -sha512 -nodes -key server.key -out server.csr -config server_request.config
```

11. Create the client certificate config file named server_cert.config. Paste the following:

```
basicConstraints = critical, CA:FALSE
keyUsage = critical, digitalSignature
```

12. Create the CA signed client certificate:

```
openssl x509 -req -sha512 -days 365 -in server.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out server.pem -extfile server_cert.config
```

13. Verify the client certificate:

```
openssl verify -verbose -CAfile rootCA.pem server.pem
```