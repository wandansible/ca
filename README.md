Ansible role: CA
================

Generate SSL certificate and private key for a host which is
signed by a locally-managed internal certificate authority
(CA). The CA certificate can be used for external software to
validate a given host is trusted and managed by ansible, or if
ownca_trust_ca is set to true, then the CA certificate will be
added to the host's trusted CA cert store.

Managing the Certificate Authority
----------------------------------

A Certificate Authority (CA) must be created to sign certificates for our ansible hosts.
Below are the commands for generating a CA key and certificate.
These are generated in a temporary directory and then copied and stored in ansible as
group variables (with private key material being protected by ansible-vault).

**Create temporary CA directory**

To start, create a temporary directory where the private key and certificate will be generated into.

```bash
ca_tmpdir=$(mktemp --directory --tmpdir ca-XXXXXXXX)
```

**Generate CA key**

Only run these commands if you are creating a new CA.
For renewing existing CAs skip to the next section.

```bash
openssl ecparam -name prime256v1 -genkey -noout -out "${ca_tmpdir}/ca.key"
cat "${ca_tmpdir}/ca.key" | ansible-vault encrypt_string --stdin-name "ownca_ca_key"
```

**Renewing existing CA certificate**

If renewing an existing CA certificate, decrypt the private key from the ansible-vault.

```bash
cat inventories/example/group_vars/all/ca.yml | yq '.ownca_ca_key' | ansible-vault decrypt
```

Put the plaintext contents of the private key into the ${ca_tmpdir}/ca.key file.

**Generate CA certificate**

```bash
openssl req -new -x509 -days 5475 -key "${ca_tmpdir}/ca.key" -out "${ca_tmpdir}/ca.cert" -subj "/CN=Ansible CA"
```

View the parsed and raw contents of the CA certificate with the following commands.

```bash
openssl x509 -text -noout -in "${ca_tmpdir}/ca.cert"
cat "${ca_tmpdir}/ca.cert"
```

Add the plaintext certificate and encrypted private key to `inventories/*/group_vars/all/ca.yml`:

* Add the contents of the ca.cert file to the `ownca_ca_cert` variable.
* Add the encrypted private key output from the ansible-vault command to the `ownca_ca_key` variable.

**Remove temporary CA directory**

When you have copied the key and certificate into your ansible inventories,
the temporary CA directory can be removed.

```bash
rm -r "${ca_tmpdir}"
```

Role Variables
--------------

```
ENTRY POINT: main - Generate SSL certificate for a host from an internal CA

        Generate SSL certificate and private key for a host which is
        signed by a locally-managed internal certificate authority
        (CA). The CA certificate can be used for external software to
        validate a given host is trusted and managed by ansible, or if
        ownca_trust_ca is set to true, then the CA certificate will be
        added to the host's trusted CA cert store.

OPTIONS (= is mandatory):

- ownca_ca_cert
        Contents of the CA certificate
        [Default: (null)]
        type: str

- ownca_ca_filename
        Filename (without extension) for the CA cert file added to the
        trusted CA cert store
        [Default: ansible-ca]
        type: str

- ownca_ca_key
        Contents of the CA key
        [Default: (null)]
        type: str

- ownca_cert_file
        Path for the generated SSL cert
        [Default: (null)]
        type: str

- ownca_common_name
        Value for the commonName (CN) field in the generated SSL cert
        [Default: (null)]
        type: str

- ownca_early_renewal
        Renew the certificate if it is not vaild from the specified
        time given as a relative time or ASN.1 timestamp, e.g. "+365d"
        for 1 year from now
        [Default: +365d]
        type: str

- ownca_generate_cert
        If true, generate an SSL certificate for the host
        [Default: True]
        type: bool

- ownca_key_curve
        When ownca_key_type = ECC, the curve used to generate the SSL
        key
        [Default: secp256r1]
        type: str

- ownca_key_file
        Path for the generated SSL key
        [Default: (null)]
        type: str

- ownca_key_size
        When ownca_key_type = RSA or DSA, size (in bits) of the
        generated SSL key
        [Default: (null)]
        type: str

- ownca_key_type
        The algorithm used to generate the SSL key
        [Default: ECC]
        type: str

- ownca_not_after
        The time, given as a relative time or ASN.1 timestamp, that
        the generated certificate should not be valid after, e.g.
        "+1825d" for 5 years from now
        [Default: +1825d]
        type: str

- ownca_san
        Subject Alternative Name (SAN) extensions added to the
        generated SSL cert
        [Default: (null)]
        elements: str
        type: list

- ownca_ssl_dir
        Directory where the generated SSL certs and keys are stored
        [Default: /etc/ssl/ansible]
        type: str

- ownca_trust_ca
        If true, add CA certificate from ownca_ca_cert to the trusted
        CA cert store
        [Default: False]
        type: bool

```

Installation
------------

This role can either be installed manually with the ansible-galaxy CLI tool:

    ansible-galaxy install git+https://github.com/wandansible/ca,main,wandansible.ca
     
Or, by adding the following to `requirements.yml`:

    - name: wandansible.ca
      src: https://github.com/wandansible/ca

Roles listed in `requirements.yml` can be installed with the following ansible-galaxy command:

    ansible-galaxy install -r requirements.yml

Example Playbook
----------------

    - hosts: all
      roles:
         - role: wandansible.ca
           become: true
           vars:
             ownca_ca_key: |
               -----BEGIN EC PRIVATE KEY-----
               XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
               -----END EC PRIVATE KEY-----
               
             ownca_ca_cert: |
               -----BEGIN CERTIFICATE-----
               XXXXXXXXXXXXXXXXXXXXXXXXXXX
               -----END CERTIFICATE-----
