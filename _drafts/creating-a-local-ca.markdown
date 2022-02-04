---
layout: post
title: 'Setting Up a Local PKI with OpenSSL'
author: Seth Forshee
tags: 
---

XXX: Maybe use footnotes?

It is sometimes useful or necessary to set up a public key infrastructure (PKI) 
for a home or small business network. For example a PKI may be used to generate 
server and client certificates for OpenVPN or to get rid of security warnings 
from your browser when accessing it's web UI.

Often there are simpler solutions to meet these needs than manually operating 
your own PKI.  For OpenVPN you can use 
[Easy-RSA](https://github.com/OpenVPN/easy-rsa), or you may be able to use a 
public certificate authority to generate TLS certificates for your router.

However, if your needs are more complex, or if you just want to gain an 
understanding of how all of this works, it is possible to run your own PKI. This 
is a complex topic that will require some study if you aren't already familiar 
with things like public key cryptography and X.509, so it's not for the faint of 
heart. Be prepared to spend some time reading materials from the 
[references](#references).

There are many, many existing sets of instructions on the Internet for using 
OpenSSL to set up a PKI, but I've found most of them to offer little explanation 
for what they're telling you to do. One of my favorites is at
[Read the Docs](http://pki-tutorial.readthedocs.io/en/latest/), which has some 
very good examples. I recommend looking there if you want to go beyond the 
simple example I provide here.

In this post, I'll provide a brief introduction to PKI and explain how to set up 
a simple PKI using OpenSSL. My goal is not to provide a script to follow, but 
rather to help you gain a baseline understanding of the process.

Note that while I had some familiarity with these topics coming in and have done 
a lot of reading while developing this post, I am not an expert on these topics.  
If you find any mistakes or confusing material, please let me know so I can make 
improvements.

# What is a Public Key Infrastructure?

PKI is a system which provides two features to computers wishing to communicate 
with each other:

- **Authentication**: This allows one computer to trust the identity of another 
  computer.
- **Encryption**: This prevents third parties from intercepting traffic passing 
  between two computers.

Perhaps the easiest way to get an idea of what this means and how it works is by 
looking at a familiar example - making a secure connection to a bank website.  
Note that this is a simplified example. The reality is somewhat more complex, 
but this will serve as a starting point that we will build upon as we walk 
though setting up a PKI.

If you wish to connect to your bank's website, you want to be sure of two 
things. First, that the website you're connecting to really belongs to your 
bank, not to someone impersonating your bank or someone sitting in between you 
and your bank who is eavesdropping on your online banking session. Second, you 
want the data to be unreadable by anyone who might be able to observe it passing 
between your computer or your bank's website.

PKI accomplishes both of these goals using
[public key cryptography](https://en.wikipedia.org/wiki/Public-key_cryptography).
Public key crypto uses two pieces of data called keys. One of these keys is kept 
secret (called the _private key_) while the other is able to be shared publicly 
(called the _public key_). The private key can be used to _sign_ data, and 
anyone with the public key is able to verify that the signature was generated 
from the exact same data using the corresponding public key. Anyone with the 
public key can also _encrypt_ data (i.e. scramble it into random gibberish), and 
that encrypted data can only be decrypted by using the private key.

When your web browser attempts to connect to a secure website, the website 
presents your browser with something called a _certificate_. The certificate 
contains various pieces of information, including some information about the 
identity of the organization which owns the website, the URL and public key for 
the website, and a signature from a third party which is already trusted by the 
web browser. The trusted third party is known as a _certificate authority_ (CA).  
Before the CA will sign the data in the certificate they verify that the 
organization is who they claim to be and that all the information in the 
certificate is valid.

Your web browser already trusts the CA and knows its public key, so once it 
verifies that the signature for the certificate is valid it can trust the 
information contained in the certificate. If the signature verification fails, 
or if the data in the certificate is for a different website, your web browser 
will warn you that someone might be trying to impersonate the website you're 
trying to connect to. Otherwise it can use the public key from the certificate 
to initiate an encrypted session with the website. (In reality public key 
encryption is only used to arrange for a shared key to be used in _symmetric key 
encryption_, which is then used to encrypt the session.)

# Terminology

Now that we have a basic idea of how PKI works, we need to establish some 
terminology that will be used when describing how to set up your own PKI. You 
will have met some of these terms already in the previous section.

- **Public Key Infrastructure** (PKI): Security architecture where public key 
  cryptography is used to convey trust using the signature of a trusted CA.
- **Certificate Authority** (CA): An entity in a PKI which issues certificates 
  and CRLs.
- **Certificate** (aka _Digital Certificate_): An electonic document used to 
  prove the ownership of a public key. In addition to information about the key 
  it includes information about the owner's identity and a digital signature 
  from a CA.
- **Certificate Revocation List** (CRL): A list of certificates which have been 
  revoked by a CA. Revoked certificates should no longer be trusted.

There are three types of CAs:

- **Root CA**: The root CA is the root of trust in a PKI. The public key of a 
  root CA is trusted implicitly and used to establish trust of other public 
  keys. Typically the root CA only issues CA certificates.
- **Intermediate CA**: Any CA below the root CA which issues certificates for 
  other CAs.
- **Signing CA**: A CA which issues certificates to users. Typically not 
  permitted to issue CA certificates.

In the simplest case the root CA would also be the signing CA, however this is 
generally not recommended. Having one or more separate signing CAs allows for 
storing the root CAs private key offline virtually all the time since it is only 
needed to sign intermediate or signing CA certificates and not user 
certificates. Adding in intermediate CAs further reduces the need to use the 
private key of the root CA in large PKIs.

# Setting Up Your Own PKI Using OpenSSL

For home and small business networks use of an intermediate CA is generally 
going to be overkill. Therefore this example will demonstrate a simple two-level 
setup with a root CA and one signing CA. The signing CA will be used to generate 
certificates for an OpenVPN server and client.

XXX: Add an image showing the PKI

Most of the work in setting up a PKI is in setting up various configuration 
files. Essentially these files contain options for various openssl commands.  
These options can also be supplied on the command line, but putting them in a 
config file helps maintain consistency and saves a lot of typing.

As we go through the config files I will explain the most important options.

These files specify the options used for various openssl commands. I will 
explain the most important options in these files, for information about the 
rest you can follow the links from
[this page](https://www.openssl.org/docs/man1.0.2/apps/) for the various openssl 
commands.

The commands below were tested using Linux with OpenSSL 1.0.2. You may need to 
make adjustments depending on your OS or OpenSSL version.

## Setting Up the Root CA

Here I will break down an example config file for a root CA section by section.
The complete configuration file is available 
[here](/assets/openssl-pki/root-ca.conf) and is a lightly modified version of 
the sample file found
[here](http://pki-tutorial.readthedocs.io/en/latest/simple/root-ca.conf.html).

The first section is the `default` section, which defines values for global 
constants. These are the values which apply to multiple openssl commands, and 
they can be referred to from the entire config file. Here, we're really just 
defining some variables for use throughout the rest of the config file. `dir` is 
the base directory working directory, which we'll just set to the current 
working directory. `ca` defines the directory which will hold our root CA files.

    [ default ]
    ca                      = root-ca               # CA name
    dir                     = .                     # Top dir

Next is the `req` section, which defines options for the `openssl req` command.  
This includes the key size, whether or not to encrypt the key, and the message 
digest to use, among others. For the key size, 2048 bits is still considered 
safe and is commonly used. However for the root CA key we'll future-proof it a 
bit by using a 4096 bit key.

If you are aware recent developments regarding the deprecation of sha1
certificates and sha1 collission attacks, you may wonder why we would want to
use sha1 here. The answer is that the root certificate is self-signed and
trusted implicitly and verification of the root certificate is not necessary.
If the certificate is never verified then it doesn't matter what message digest 
we use. It will matter for later certificates which are signed by the root CA or 
a subordinate CA.

The values for `distinguished_name` and `req_extensions` refer to later sections 
in the config file.

    [ req ]
    default_bits            = 4096                  # RSA key size
    encrypt_key             = yes                   # Protect private key
    default_md              = sha1                  # MD to use
    utf8                    = yes                   # Input is UTF-8
    string_mask             = utf8only              # Emit UTF-8 strings
    prompt                  = no                    # Don't prompt for DN
    distinguished_name      = ca_dn                 # DN section
    req_extensions          = ca_reqext             # Desired extensions

Now we'll define the `ca_dn` section which was referred to in the `req` section.  
These should be change to values appropriate for your use case.

    [ ca_dn ]
    0.domainComponent       = "org"
    1.domainComponent       = "simple"
    organizationName        = "Simple Inc"
    organizationalUnitName  = "Simple Root CA"
    commonName              = "Simple Root CA"

We also need to define the `ca_reqext` section referred to by the `req` section.  
This section is very important, as it defines the ways in which the root CA key 
can be used. The use of `critical` in the values indicates that systems should 
reject the certificate if it does not recognize the extension.

The root CA certificate is used for two purposes, signing other certificates and 
signing certificate revocation lists (CRLs). This is done by asserting
`keyCertSign` and `cRLSign` in the `keyUsage` field.

Whenever `keyCertSign` is asserted in `keyUsage`, `CA` must also be asserted in 
`basicConstraints` (as only a CA signs other certificates). `pathlen` specifies 
how many intermediate CA certificates are allowed to follow this certificate; 
for our purposes a value of `1` is appropriate.

The `subjectKeyIdentifier` is required whenever `CA` is `true` and should be set 
to `hash`.

    [ ca_reqext ]
    keyUsage                = critical,keyCertSign,cRLSign
    basicConstraints        = critical,CA:true
    subjectKeyIdentifier    = hash

Next we set up configuration for the `oppenssl ca` command. The first section 
just specifies which section defines paramaters for the default CA.

    [ ca ]
    default_ca              = root_ca               # The default CA section

Then we define the `root_ca` section with the root CA parameters. Some of the 
more important fields:

- `unique_subject`: This specifies whether or not certificates in the database 
  are required to have unique subjects. Setting this to `yes` will prevent 
  accidentally assigning to certificates with the same subject, but it also 
  prevents issuing new certificates prior to expiration of the old certificate, 
  so this is set to `no` here.
- `default_md`: This defines the default message digest for certificates created 
  with the root CA. Since these certificates will be verified, we will use 
  `sha256`.
- `x509_extensions`: Defines the extensions added to certificates issued by the 
  root CA. This refers to a later section, which will specify the extensions 
  used for a signing CA.

Other extensions are explained in the OpenSSL documentation.

    [ root_ca ]
    certificate             = $dir/ca/$ca.crt       # The CA cert
    private_key             = $dir/ca/$ca/private/$ca.key # CA private key
    new_certs_dir           = $dir/ca/$ca           # Certificate archive
    serial                  = $dir/ca/$ca/db/$ca.crt.srl # Serial number file
    crlnumber               = $dir/ca/$ca/db/$ca.crl.srl # CRL number file
    database                = $dir/ca/$ca/db/$ca.db # Index file
    unique_subject          = no                    # Require unique subject
    default_days            = 3652                  # How long to certify for
    default_md              = sha256                # MD to use
    policy                  = match_pol             # Default naming policy
    email_in_dn             = no                    # Add email to cert DN
    preserve                = no                    # Keep passed DN ordering
    name_opt                = ca_default            # Subject DN display options
    cert_opt                = ca_default            # Certificate display options
    copy_extensions         = none                  # Copy extensions from CSR
    x509_extensions         = signing_ca_ext        # Default cert extensions
    default_crl_days        = 365                   # How long before next CRL
    crl_extensions          = crl_ext               # CRL extensions

Now we need to define the naming policy identified above. This specifies which 
components of the domain name are included in certificates, which must match the 
root CA's certificate, and which are required to be supplied.

    [ match_pol ]
    domainComponent         = match                 # Must match 'simple.org'
    organizationName        = match                 # Must match 'Simple Inc'
    organizationalUnitName  = optional              # Included if present
    commonName              = supplied              # Must be present

The next sections specify what types of certificates our root CA will be able to 
create. There's one section for the root CA certificate and another for signing 
CA certificates. The name of the extensions section can be passed to `openssl 
ca` via the `-extensions` argument.

    [ root_ca_ext ]
    keyUsage                = critical,keyCertSign,cRLSign
    basicConstraints        = critical,CA:true
    subjectKeyIdentifier    = hash
    authorityKeyIdentifier  = keyid:always
    
    [ signing_ca_ext ]
    keyUsage                = critical,keyCertSign,cRLSign
    basicConstraints        = critical,CA:true,pathlen:0
    subjectKeyIdentifier    = hash
    authorityKeyIdentifier  = keyid:always

Finally we need to specify the CRL extenstions, which specifies that the subject 
key identifier should be copied from the parent and that an error is returned if 
this option fails.

    [ crl_ext ]
    authorityKeyIdentifier  = keyid:always

With the configuration file complete, we can now create our root CA files. We 
start by creating the directories specified in our configuration file, and 
limiting access to the directory containing the private key to only the current 
user.

    mkdir -p ca/root-ca/private ca/root-ca/db crl certs
    chmod 700 ca/root-ca/private

Then we must create empty database files before we can use the `openssl ca` 
command.

    touch ca/root-ca/db/root-ca.db
    touch ca/root-ca/db/root-ca.db.attr

Next we create the private key and certificate signing request using the 
configuration from our file.

    openssl req -new -config root-ca.conf \
        -out ca/root-ca.csr \
        -keyout ca/root-ca/private/root-ca.key

Then we issue a root CA certificate based on the CSR we just created.

    openssl ca -selfsign -config root-ca.conf \
        -in ca/root-ca.csr -out ca/root-ca.crt \
        -extensions root_ca_ext -create_serial

'-create-serial` is needed only for the first certificate signed with each CA, 
to initialize the file which contains the next serial number with a random 
number. It only does this when the file is not already present though, so it 
doesn't do any harm to supply this option after the file has already been 
created.

## Setting Up the Signing CA

As with the root CA, most of the work here is in setting up the signing CA 
configuration file. The full configuration can be found
[here](/assets/openssl-pki/signing-ca.conf), and is a lightly modified version 
of the sample file found
[here](http://pki-tutorial.readthedocs.io/en/latest/simple/signing-ca.conf.html).  
I'll avoid redundant explanations of configuration options and only focus on 
options which are new or different from the root CA.

The first few sections don't look much different than those for the root CA. The 
most important difference is that we use `sha256` for `default_md` since this 
certificate is not self signed. As with the root CA, you should change the 
values in the `ca_dn` section to something appropriate for your use.

    [ default ]
    ca                      = signing-ca            # CA name
    dir                     = .                     # Top dir
    
    [ req ]
    default_bits            = 2048                  # RSA key size
    encrypt_key             = yes                   # Protect private key
    default_md              = sha1                  # MD to use
    utf8                    = yes                   # Input is UTF-8
    string_mask             = utf8only              # Emit UTF-8 strings
    prompt                  = no                    # Don't prompt for DN
    distinguished_name      = ca_dn                 # DN section
    req_extensions          = ca_reqext             # Desired extensions
    
    [ ca_dn ]
    0.domainComponent       = "org"
    1.domainComponent       = "simple"
    organizationName        = "Simple Inc"
    organizationalUnitName  = "Simple Signing CA"
    commonName              = "Simple Signing CA"

The `ca_reqext` has one difference from the same section for the root CA. The 
`basicConstraints` contain `pathlen:0`. `pathlen` specifies how many 
intermediate CA certificates are allowed to follow this certificate. A signing 
certificate should not be used to sign another intermeddiate certificate, so a 
value of `0` enforces this.

    [ ca_reqext ]
    keyUsage                = critical,keyCertSign,cRLSign
    basicConstraints        = critical,CA:true,pathlen:0
    subjectKeyIdentifier    = hash

The `ca` section looks largely the same as well, aside from some name changes 
and shortening the periods for `default_days` and `default_crl_days`.

    [ ca ]
    default_ca              = signing_ca            # The default CA section
    
    [ signing_ca ]
    certificate             = $dir/ca/$ca.crt       # The CA cert
    private_key             = $dir/ca/$ca/private/$ca.key # CA private key
    new_certs_dir           = $dir/ca/$ca           # Certificate archive
    serial                  = $dir/ca/$ca/db/$ca.crt.srl # Serial number file
    crlnumber               = $dir/ca/$ca/db/$ca.crl.srl # CRL number file
    database                = $dir/ca/$ca/db/$ca.db # Index file
    unique_subject          = no                    # Require unique subject
    default_days            = 730                   # How long to certify for
    default_md              = sha256                # MD to use
    policy                  = match_pol             # Default naming policy
    email_in_dn             = no                    # Add email to cert DN
    preserve                = no                    # Keep passed DN ordering
    name_opt                = ca_default            # Subject DN display options
    cert_opt                = ca_default            # Certificate display options
    copy_extensions         = copy                  # Copy extensions from CSR
    x509_extensions         = openvpn_client_ext    # Default cert extensions
    default_crl_days        = 7                     # How long before next CRL
    crl_extensions          = crl_ext               # CRL extensions
    
    [ match_pol ]
    domainComponent         = match                 # Must match 'simple.org'
    organizationName        = match                 # Must match 'Simple Inc'
    organizationalUnitName  = optional              # Included if present
    commonName              = supplied              # Must be present
    
    [ any_pol ]
    domainComponent         = optional
    countryName             = optional
    stateOrProvinceName     = optional
    localityName            = optional
    organizationName        = optional
    organizationalUnitName  = optional
    commonName              = optional
    emailAddress            = optional

Now we add a couple of new extension sets for OpenVPN, one for the server and 
another for clients. These are identical except for `extendedKeyUsage`, where 
the server needs `serverAuth` and the client needs `clientAuth`.

If you intend to sign other types of certificates, you will need to define 
extension sets with values appropriate for those certificates' uses.

    [ openvpn_server_ext ]
    keyUsage                = critical,digitalSignature,keyEncipherment
    basicConstraints        = CA:false
    extendedKeyUsage        = serverAuth
    subjectKeyIdentifier    = hash
    authorityKeyIdentifier  = keyid:always
    
    [ openvpn_client_ext ]
    keyUsage                = critical,digitalSignature,keyEncipherment
    basicConstraints        = CA:false
    extendedKeyUsage        = clientAuth
    subjectKeyIdentifier    = hash
    authorityKeyIdentifier  = keyid:always
    
    [ crl_ext ]
    authorityKeyIdentifier  = keyid:always


# <a name="references"></a>References






# SAF for later

    mkdir -p ca/signing-ca/private ca/signing-ca/db crl certs
    chmod 700 ca/signing-ca/private
    touch ca/signing-ca/db/signing-ca.db
    touch ca/signing-ca/db/signing-ca.db.attr
    openssl req -new -config signing-ca.conf \
        -out ca/signing-ca.csr \
        -keyout ca/signing-ca/private/signing-ca.key
    openssl ca -config root-ca.conf \
        -in ca/signing-ca.csr -out ca/signing-ca.crt \
        -extensions signing_ca_ext


# SAF Full command list

mkdir -p ca/root-ca/private ca/root-ca/db crl certs
chmod 700 ca/root-ca/private
touch ca/root-ca/db/root-ca.db
touch ca/root-ca/db/root-ca.db.attr
openssl req -new -config root-ca.conf -out ca/root-ca.csr -keyout ca/root-ca/private/root-ca.key
openssl ca -selfsign -config root-ca.conf -in ca/root-ca.csr -out ca/root-ca.crt -extensions root_ca_ext -create_serial
mkdir -p ca/signing-ca/private ca/signing-ca/db crl certs
chmod 700 ca/signing-ca/private
touch ca/signing-ca/db/signing-ca.db
touch ca/signing-ca/db/signing-ca.db.attr
openssl req -new -config signing-ca.conf -out ca/signing-ca.csr -keyout ca/signing-ca/private/signing-ca.key
openssl ca -config root-ca.conf -in ca/signing-ca.csr -out ca/signing-ca.crt -extensions signing_ca_ext
openssl req -new -config openvpn-server.conf -out certs/vpn.forshee.me.csr -keyout certs/vpn.forshee.me.key
openssl ca -config signing-ca.conf -in certs/vpn.forshee.me.csr -out certs/vpn.forshee.me.crt -extensions openvpn_server_ext  -policy openvpn_server_pol -create_serial
openssl req -new -config openvpn-client.conf -out certs/ubuntu-xps13.csr -keyout certs/ubuntu-xps13.key
openssl ca -config signing-ca.conf -in certs/ubuntu-xps13.csr -out certs/ubuntu-xps13.crt -extensions openvpn_client_ext -policy openvpn_client_pol
# Generate CRL
openssl ca -config signing-ca.conf -gencrl -out crl/signing-ca.crl


SAF: For many purposes we need a "chained" file with both the root CA and signing CA certifictes. To create this:

openssl x509 -in ca/signing-ca.crt -outform pem -out ca/ca-chain.crt
openssl x509 -in ca/root-ca.crt -outform pem >> ca/ca-chain.crt










Perhaps the easiest way to understand what PKI does is through a familiar 
example of how it is used. Say you want to visit the website of your bank. You 
want to be sure that the website you connect to is really that of your bank, not 
another website pretending to be your bank or one sitting in between you and 
your bank eavesdropping on your online banking session.

Your web browser relies to PKI to ensure that the website you're connecting to 
really does belong to your bank. Basically, your bank pays a trusted third party 
to verify that they are who they claim to be and that the website belongs to 
them. Then when you connect to the website it provides to your web browser the 
evidence that they are trusted by the third party. Since your web browser also 
trusts that third party it trusts that you're really connecting to your bank and 
not someone else. However, if the evidence presented by the website doesn't 
check out your browser gives you some nasty warnings that the website might not 
be who it claims to be.

This is accomplished using
[public key cryptography](https://en.wikipedia.org/wiki/Public-key_cryptography),
which uses a pair of keys, one of which must be kept secret (the private key) 
and another which can be shared (the public key). The trusted third party (known 
as a certificate authority or CA) uses its private key to sign the website's 
public key. Then the web browser already tursts the public key of the CA, so it 
can use it to verify that the signature on the website's public key is 
authentic. If the signature is authentic the web browser trusts the identity of 
the website.
