### 1. Download CFSSL and CFSSLJSON

```
wget -q --show-progress --https-only --timestamping \
    https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssl \
    https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssljson
```

### 2. Add executable permission to them

```
chmod +x cfssl cfssljson
```

### 3. move these exe files to /usr/local/bin directory

```
sudo mv cfssl cfssljson /usr/local/bin/
```
### 4. What is the use of cfssljson ?
The cfssljson program, which takes the JSON output from the cfssl programs and writes certificates, keys, CSRs, and bundles to disk.

You can create a brand new CA with CFSSL using the -initca option. As we saw above, this takes a JSON blob with the certificate request, and creates a new CA key and certificate:

```
$ cfssl gencert -initca ca_csr.json
```

This will return a CA certificate and CA key that is valid for signing. The CA is meant to function as an internal tool for creating certificates. 

### 5. How to check the supported values for csr and config ?

    $ cfssl print-defaults config

    ```{
        "signing": {
            "default": {
                "expiry": "168h"
            },
            "profiles": {
                "www": {
                    "expiry": "8760h",
                    "usages": [
                        "signing",
                        "key encipherment",
                        "server auth"
                    ]
                },
                "client": {
                    "expiry": "8760h",
                    "usages": [
                        "signing",
                        "key encipherment",
                        "client auth"
                    ]
                }
            }
        }
    }```

    $ cfssl print-defaults csr

    ```{
        "CN": "example.net",
        "hosts": [
            "example.net",
            "www.example.net"
        ],
        "key": {
            "algo": "ecdsa",
            "size": 256
        },
        "names": [
            {
                "C": "US",
                "ST": "CA",
                "L": "San Francisco"
            }
        ]
    }```

#### What is a CSR? 

A CSR or Certificate Signing request is a block of encoded text that is given to a Certificate Authority when applying for an SSL Certificate. It is usually generated on the server where the certificate will be installed and contains information that will be included in the certificate such as the organization name, common name (domain name), locality, and country. It also contains the public key that will be included in the certificate. A private key is usually created at the same time that you create the CSR, making a key pair.

#### What is contained in a CSR?
Name	                    Explanation	                                Examples
| :-------------------------|:----------------------------------------:| --------------------------:|
Common Name	                FQDN of your server.                        *.google.com
                            This must match exactly what you            mail.google.com
                            type in your web browser or you will        
                            receive a name mismatch error.	
|                           |                                          |
Organization	            The legal name of your organization.        Google Inc.
                            This should not be abbreviated and should 
                            include suffixes such as Inc, Corp, or LLC.	
                           
|                           |                                          |
Organizational Unit	        The division of your organization handling  Information Technology
                            the certificate.	                        IT Department

|                           |                                          |
City/Locality	            The city where your organization is         Mountain View 
                            located.

|                           |                                          |    
State/County/Region	        The state/region where your organization    California
                            is located. This shouldn't be abbreviated.	

|                           |                                          |                
Country	                    The two-letter ISO code for the country     IN
                            where your organization is location.	    US

|                           |                                          |
Email address	            An email address used to contact your       webmaster@google.com
                            organization.	

|                           |                                          | 
Public Key	                The public key that will go into the certificate.	
                            The public key is created automatically

### 6. Defining few terms used in cfssl 

    sign             signs a certificate
    bundle           build a certificate bundle
    genkey           generate a private key and a certificate request
    gencert          generate a private key and a certificate
    serve            start the API server
    version          prints out the current version
    selfsign         generates a self-signed certificate
    print-defaults   print default configurations

### 7. Generating a certificate authority [CA]

In this section you will provision a Certificate Authority that can be used to generate additional TLS certificates. This is the Root CA for our environment

1. We'll Create a ca-config.json with signing and profile details. 
This will be used to create server or client certificates that can be used to set up SSL/TSL based authentication.

2. Create a ca-csr.json file

3. Use Cfssljson to take json output and write certificates, keys, CSRs to disk

```
{
    cat > ca-config.json << EOF
    {
    "signing": {
    "default": {
    "expiry": "8760h"
    },
    "profiles": {
    "kubernetes": {
    "usages": ["signing", "key encipherment", "server auth", "client auth"],
    "expiry": "8760h"
    }
    }
    }
    }
    EOF
    cat > ca-csr.json << EOF
    {
    "CN": "Kubernetes",
    "key": {
    "algo": "rsa",
    "size": 2048
    },
    "names": [
    {
    "C": "US",
    "L": "Portland",
    "O": "Kubernetes",
    "OU": "CA",
    "ST": "Oregon"
    }
    ]
    }
    EOF
    cfssl gencert -initca ca-csr.json | cfssljson -bare ca
}
```

OUTPUT:
    [INFO] generating a new CA key and certificate from CSR
    [INFO] generate received request
    [INFO] received CSR
    [INFO] generating key: rsa-2048
    [INFO] encoded CSR
    [INFO] signed certificate with serial number 575514558967771581279537545623874943296973655847
#### These below are the three files generated

ca-key.pem :    Root CA Private key. Keep it very safe!!
ca.pem :   Root CA Public key
ca.csr :    Certificate Signing Request


Now we have our root CA which is the most important file. The root CA will allow us to generate intermediate certificates. Intermediate certificates can be used just like the CA to generate other intermediate certificates or to directly sign certificates and keys.

### VERIFY
openssl x509 -in /<PATH>/certs ca.pem -text -noout