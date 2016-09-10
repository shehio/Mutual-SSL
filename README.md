# Mutual-SSL
## Motivation: 
 
* Why do we need ssl/tls?  
Integrity, and confidentiality. In other words: What being said isn’t tampered (by a man in the middle) with, and to ensure that a party is really who he is claiming to be. 
 
* What’s the difference between ssl/tls and https?  
SSL(secure socket layer) or TLS (transport layer security) operate on the transport layer in the OSI model, they can work with http or any other compatible protocol, https is http + ssl/tls. 
 
* What’s the difference between ssl and tls?  
TLS is the successor of SSL. 
 
* What’s the difference between one-way ssl and two-way (mutual) ssl?   
In one way ssl, only the authentication of the server is being verified, while in mutual ssl the authenticity of the two parties are. 
 
* What’s the difference between keystore and trust store?  
The key store saves your private keys, while the trust store save the certificates from parties you trust. 
 
* What’s the difference between a public key, and a certificate?  
A certificate contains a public key. The certificate, in addition to the public key, contains additional information, such as issuer, what it's supposed to be used for, and any other type of metadata. Typically a certificate is itself signed with a private key that verifies its authenticity.

## Notes: 
By default, a key pair is generated and (encrypted by a passphrase), and then the public key is extracted from that file by the –pubout option for openssl.

## Assumptions: 
I am using a mac, having openssl installed.

### Setting up our keys:
Let’s work in an empty directory (say: ~/ssl_keys)  

* Create a self-signed certificate authority (a private key pair):
 
``` 
openssl req -newkey rsa:2048 -nodes -keyform PEM -keyout CA.key -x509 -days 365 -outform PEM -out CA.crt 
```

### options meaning:

*req* : certificate generating command in openssl.  
*nodes* : if this option is specified then if a private key is created it will not be encrypted.  
*keyform* : either PEM or DER.  
*x509* : this option outputs a self signed certificate instead of a certificate request. This is typically used to generate a test certificate or a self signed root CA.  It has properties like:  
**days** until expiration.  
**outform** either PEM or DER.  

```
Generating a 2048 bit RSA private key 
..........................................+++ 
.....................+++ 
writing new private key to 'CA.key' 
```  

``` 
You are about to be asked to enter information that will be incorporated 
into your certificate request. 
What you are about to enter is what is called a Distinguished Name or a DN. 
There are quite a few fields but you can leave some blank 
For some fields there will be a default value, 
If you enter '.', the field will be left blank. 
 
Country Name (2 letter code) [AU]:US 
State or Province Name (full name) [Some-State]:New York 
Locality Name (eg, city) []:New York 
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Evil 
Organizational Unit Name (eg, section) []: 
Common Name (e.g. server FQDN or YOUR name) []:shehio 
Email Address []:shehabyasser@gmail.com 
```  

* Generate a server key pair  

``` 
openssl genrsa -out server.key 2048 
```  

* Generate a request file (to be signed by the CA):  (I omitted the previous prompt repeating here.)  

``` 
openssl req -new -key server.key -out server.req 
```  

* See what we’ve got now:  

``` 
ls  
CA.crtCA.keyserver.keyserver.req 
```  

* Actually sign the server key by the CA:  

```
openssl x509 -req -in server.req -CA CA.crt -CAkey CA.key -set_serial 0x3E8  -extensions server -days 365 -outform PEM -out server.crt 
```  

* See what we’ve got now:  

``` 
ls
CA.crt  CA.key server.crt  server.key  server.req  
```  

* Do the same for the client as for the server, create the key:  

``` 
openssl genrsa -out client.key 2048 
```  

* Create the request file:  

``` 
openssl req -new -key client.key -out client.req 
```

* See what we’ve got:  

``` 
ls 
CA.crt  CA.key  client.crt  client.key  client.req  server.crt  server.key  server.req 
```  

* Generate the p12 (or pkcs12) file (compact file containing both the private key, and CA): 

``` 
openssl pkcs12 -export -inkey client.key -in client.crt -out client.p12 
Enter Export Password: 
Verifying - Enter Export Password:
```  

* See what we’ve got:  

``` 
ls 
CA.crt  CA.key  client.crt  client.key  client.p12  client.req  server.crt  server.key  server.req 
```  

## Spinning up an EC2 instance: 

* launch a free tier EC2 instance on AWS, choose ubuntu as the OS, and open at least ports: 22 for ssh, 80 for http, 443 for https from anywhere.  
* Don’t forget to tag your instance, for easier recognition.  
* Create a key pair or choose an existing key.  
* Log in (to user ubuntu and by the key you have/downloaded).  


```` 
ssh –i key.pem ubuntu@public_ip 
```
  
## Install and configure Apache2: 
 
* After logging in: elevate your privileges: 
``` 
Sudo su 
``` 
* Update your local repos 
 ``` 
apt-get update 
``` 
* Install apache2 package for apache server on Ubuntu: 
``` 
apt-get install -y apache2 
``` 
* Now you can access your public ip from the browser and it will load apache’s default web page. 
 
## Configuring Apache for https (ssl): 

* Change the directory to apache’s home  

```
Cd /etc/apache2/  
```  

* Create a symbolic link for the https virtual host (remember to use the absolute path, and not relative path or it will fail later on):  
 
``` 
ln –s /etc/apache2/sites-available/default-ssl.conf /etc/apahce2/sites-enabled/default-ssl.conf 
```  

* And linking ssl modules as well.  

``` 
ln -s /etc/apache2/mods-available/socache_shmcb.load /etc/apache2/mods-enabled/socache_shmcb.load 
ln -s /etc/apache2/mods-available/ssl.conf /etc/apache2/mods-enabled/ssl.conf 
ln -s /etc/apache2/mods-available/ssl.load /etc/apache2/mods-enabled/ssl.load 
```  

* Restart apache:  

``` 
service apache2 restart 
```  

Now you can access your public ip from the browser by HTTPS (though you’ll have the insecure page) and it will load apache’s default web page. 
 
## A moment here: 
We enabled apache to listen on port 443, by linking the default-ssl.conf file into the enabled sites. If we quickly go through it, we will find that it uses *ubuntu’s default self signed certificate* located in  /etc/ssl/certs the default directory for certificates in ubuntu.  

``` 
SSLCertificateFile/etc/ssl/certs/ssl-cert-snakeoil.pem 
SSLCertificateKeyFile /etc/ssl/private/ssl-cert-snakeoil.key 
``` 
 
If you list the directory you’ll find a lot of certificates trusted by ubuntu by default, whilst if you list /etc/ssl/private you’ll only find the snakeoil key. Reasonably the directory, it only has one key, the dummy one generated by ubuntu for trivial testing (exactly what we are doing).  
 
If you foolishly deleted them, or you just want to get the unsafe screen again in your browser, you can regenerate them by executing the shell script: 
 
```
make-ssl-cert generate-default-snakeoil --force-overwrite  
```  

## Configuring Apache for mutual ssl: 
* Copy the server key pair and the CA to our ec2 instance.  
Note: We don’t need the requests any more, and even the private key for the certificate (we only used it to sign new certificates).  
I like to use the app cyberduck to transfer my keys around, otherwise use secure copy using your ec2 instance key.  
* Change the keys, and the certs, for apache: 
For starters let's use the root certificate, by it’s own without using any certificates chain. Actually to use the Root CA’s certificate, we’ll need it’s private key, so copy it as well. 
Note: I copied the keys into the home directory for ubuntu.  
 
* Modify those properties into the sites-available:  
 
``` 
SSLCertificateFile/home/ubuntu/CA.crt 
SSLCertificateKeyFile /home/ubuntu/CA.key 
``` 
* And as always, restart the server:

```
service apache2 restart  
```

We can trust the address from the web browser, if we add the CA.crt from the web browser (for chrome this can’t be trusted anyway at least because it’s using sha1: more on that in the reference) 
 
* Now, let’s use https (ssl) using the server key, experiencing the certificate chaining here by modifying the same properties again in /etc/apache2/sites-enabled/default-ssl.conf, and add the CA:  

``` 
SSLCertificateFile/home/ubuntu/server.crt 
SSLCertificateKeyFile/home/ubuntu/server.key 
SSLCACertificateFile /home/ubuntu/CA.crt 
```  

* Restart the server as usual:  

``` 
service apache2 restart 
```  

Then give it a visit from your browser, and shall be working again.  
 
### Enabling Mutual Authentication:
 
*  Enable client authentication: 
``` 
SSLVerifyClient require 
SSLVerifyDepth  1 
SSLCACertificateFile /home/ubuntu/CA.crt 
``` 
* And then restart the server 
service apache2 restart 
 
 
This shan’t be reachable from unless you import the p12 file to your browser, you can do so in chrome from settings and in your certificates, you can import client.p12 file. 
 
### How does this fit into Java? 
 
### One way ssl: 
 
Equivalently to trusting the CA certificate, in a web browser, we’ll have to add the CA certificate into the trust store:  
``` 
keytool -import -alias CA -file CA.crt -keystore truststore  
``` 
 
Mutual SSL: 
Equivalent to adding the p12 file to the certificates in the browser: import the cert to your keystore to be able to use it in java. 
```
keytool -importkeystore -destkeystore keystore.jks -srckeystore client.p12 -srcstoretype pkcs12 
```
Index: 
In cryptography, X.509 is an important standard for a public key infrastructure (PKI) to manage digital certificates[1] and public-key encryption and a key part of the Transport Layer Security protocol used to secure web and email communication 
 
A public key infrastructure (PKI) is a set of roles, policies, and procedures needed to create, manage, distribute, use, store, and revoke digital certificates[1] and manage public-key encryption. The purpose of a PKI is to facilitate the secure electronic transfer of information for a range of network activities such as e-commerce, internet banking and confidential email. 
 
Difference between PEM and DER is that the first encodes  in base64 and encrypts it, while the latter encodes in binary, the default format in openssl is PEM. 
 
PKCS#12 or PFX format is a binary format for storing the server certificate, any intermediate certificates, and the private key into a single encryptable file.  
 
The default trust store for java is for java 8, and on mac.   

``` 
/Library/Java/JavaVirtualMachines/jdk1.8.0_74.jdk/Contents/Home/jre/lib/security/cacerts 
``` 
note that:  
``` 
JAVAHOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_74.jdk/Contents/Home
``` 

## References: 
*[Certificates in ubuntu](https://help.ubuntu.com/lts/serverguide/certificates-and-security.html) 
*[How do I create a self-signed SSL certificate?](http://askubuntu.com/questions/49196/how-do-i-create-a-self-signed-ssl-certificate) 
*[Certificates and Public Keys](https://msdn.microsoft.com/en-us/library/windows/desktop/aa376502(v=vs.85).aspx) 
*[https://www.openssl.org/docs/manmaster/apps/req.html](https://www.openssl.org/docs/manmaster/apps/req.html) 
*[2 Way SSL with example](https://prasenjitdas235.blogspot.com/2014/11/2-way-ssl-with-example.html?showComment=1471532549494#c769944397867855572) 
*[Importing the private-key/public-certificate pair in the Java KeyStore](http://stackoverflow.com/questions/17695297/importing-the-private-key-public-certificate-pair-in-the-java-keystore) 
*[Why does Chrome on 1 computer say my certificate is invalid/insecure?](http://serverfault.com/questions/684736/why-does-chrome-on-1-computer-say-my-certificate-is-invalid-insecure) 

