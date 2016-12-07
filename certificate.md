# Certificate

## Certificate Authority (CA)

### Create the Root Key
The first step is to create the private root key which only takes one step. In the example below, I’m creating a 2048 bit key:

```
openssl genrsa -out rootCA.key 2048
``` 
Keep this private key very private. This is the basis of all trust for your certificates, and if someone gets a hold of it, they can generate certificates that your browser will accept. You can also create a key that is password protected by adding -des3:
```
openssl genrsa -des3 -out rootCA.key 2048
```

### Self-sign the CA-certificate
```
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 1024 -out rootCA.pem
```
This will open up an interactive shell, just enter your information
```
Country Name (2 letter code) [AU]: SE
State or Province Name (full name) [Some-State]:.
Locality Name (eg, city) []: Norrkoping
Organization Name (eg, company) [Internet Widgits Pty Ltd]: Snorbuse
Organizational Unit Name (eg, section) []: kubernetes
Common Name (eg, YOUR name) []: Snorbuse kubernetes cluster
# Common Name (eg, YOUR name) []:Data Center Overlords
Email Address []:.
```

```
Subject: C=DE, ST=Germany, L=City, O=Company, OU=Organisation-Unit, CN=server1.example.com/emailAddress=alias@example.com
```

You should now hava a `rootCA.pem` file and that's your private certficate authority (CA). You can verify the information with
```
openssl x509 -in rootCA.pem -noout -text
```

### Create a certificate
Now it's time to create a certficiate for your device/system/host. First, create a private key for your system. 
```
openssl genrsa -out device.key 2048
```

Once the key is created, you’ll generate the certificate signing request.
```
openssl req -new -key device.key -out device.csr
```
You’ll be asked various questions (Country, State/Province, etc.). Answer them how you see fit. The important question to answer though is common-name.
```
Common Name (eg, YOUR name) []: 10.0.0.1
```
Whatever you see in the address field in your browser when you go to your device must be what you put under common name, even if it’s an IP address.  Yes, even an IP (IPv4 or IPv6) address works under common name. If it doesn’t match, even a properly signed certificate will not validate correctly and you’ll get the “cannot verify authenticity” error. 

Once that’s done, you’ll sign the CSR, which requires the CA root key.
```
echo "subjectAltName = DNS:worker2, IP:10.32.0.1, IP:10.240.0.10, IP:10.240.0.11, IP:10.240.0.12, IP:10.240.0.20, IP:10.240.0.21, IP:10.240.0.22, IP:127.0.0.1" > extfile.cnf
openssl x509 -req -in device.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out device.crt -days 500 -sha256 -extfile extfile.cnf
```

## Dictionary<sup>1</sup>

### Common
* **.csr** This is a Certificate Signing Request. Some applications can generate these for submission to certificate-authorities. The actual format is PKCS10 which is defined in RFC 2986. It includes some/all of the key details of the requested certificate such as subject, organization, state, whatnot, as well as the public key of the certificate to get signed. These get signed by the CA and a certificate is returned. The returned certificate is the public certificate (which includes the public key but not the private key), which itself can be in a couple of formats.
* **.pem** Defined in RFC's 1421 through 1424, this is a container format that may include just the public certificate (such as with Apache installs, and CA certificate files /etc/ssl/certs), or may include an entire certificate chain including public key, private key, and root certificates. Confusingly, it may also encode a CSR (e.g. as used here) as the PKCS10 format can be translated into PEM. The name is from Privacy Enhanced Mail (PEM), a failed method for secure email but the container format it used lives on, and is a base64 translation of the x509 ASN.1 keys. PEM on it's own isn't a certificate, it's just a way of encoding data. X.509 certificates are one type of data that is commonly encoded using PEM.
* **.key** This is a PEM formatted file containing just the private-key of a specific certificate and is merely a conventional name and not a standardized one. In Apache installs, this frequently resides in /etc/ssl/private. The rights on these files are very important, and some programs will refuse to load these certificates if they are set wrong.
* **.pkcs12 .pfx .p12** Originally defined by RSA in the Public-Key Cryptography Standards, the "12" variant was enhanced by Microsoft. This is a passworded container format that contains both public and private certificate pairs. Unlike .pem files, this container is fully encrypted. Openssl can turn this into a .pem file with both public and private keys: openssl pkcs12 -in file-to-convert.p12 -out converted-file.pem -nodes 

### Uncommon
* **.der** A way to encode ASN.1 syntax in binary, a .pem file is just a Base64 encoded .der file. OpenSSL can convert these to .pem (openssl x509 -inform der -in to-convert.der -out converted.pem). Windows sees these as Certificate files. By default, Windows will export certificates as .DER formatted files with a different extension. Like...
* **.cert .cer .crt** A .pem (or rarely .der) formatted file with a different extension, one that is recognized by Windows Explorer as a certificate, which .pem is not.

### Summary
**PEM** Governed by RFCs, it's used preferentially by open-source software. It can have a variety of extensions (.pem, .key, .cer, .cert, more)
**DER** The parent format of PEM. It's useful to think of it as a binary version of the base64-encoded PEM file. Not routinely used by much outside of Windows.


## Links
http://serverfault.com/questions/9708/what-is-a-pem-file-and-how-does-it-differ-from-other-openssl-generated-key-file/9717#9717
http://www.akadia.com/services/ssh_test_certificate.html
https://datacenteroverlords.com/2012/03/01/creating-your-own-ssl-certificate-authority/
http://blog.endpoint.com/2014/10/openssl-csr-with-alternative-names-one.html
http://wiki.cacert.org/FAQ/subjectAltName

The Most Common OpenSSL Commands
https://www.sslshopper.com/article-most-common-openssl-commands.html