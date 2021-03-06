# LDAP PASSWORDLESS AUTHENTICATION

## Prerequisite

Note for configuring your system as the Ldap server and Ldap client refer to the [Ldap configuration](https://github.com/debeabhijeet/LDAP).

After setting up the Ldap server and client follow the following steps for passwordless authentication.

First we need to add the openssh schema to the **LDAP-Server**, So do the Following:

```shell
vi /etc/openldap/schema/openssh-ldap.ldif

dn: cn=openssh-lpk,cn=schema,cn=config
objectClass: olcSchemaConfig
cn: openssh-lpk
olcAttributeTypes: ( 1.3.6.1.4.1.24552.500.1.1.1.13 NAME 'sshPublicKey'
    DESC 'MANDATORY: OpenSSH Public key'
    EQUALITY octetStringMatch
    SYNTAX 1.3.6.1.4.1.1466.115.121.1.40 )
olcObjectClasses: ( 1.3.6.1.4.1.24552.500.1.1.2.0 NAME 'ldapPublicKey' SUP top AUXILIARY
    DESC 'MANDATORY: OpenSSH LPK objectclass'
    MAY ( sshPublicKey $ uid )
    )
```
Now Run the following command to include the schema in the ldap.

```shell
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/openssh-ldap.ldif
```

Note that all the below things need to be done on **LDAP-Client** only.

## Instalation of program

Now understand that to do passwordless authentication the ldap client needs to fetch the ssh-key from the ldap server and to do this we require some kind of script or program. Therefore to help with this challenge we will use python program.

```shell
zypper install python3  python3-pip python3-devel  openldap2-devel  gcc 

python3 -m pip install pyldap

python3 -m pip install ssh-ldap-pubkey
```

After installing the above things give the ownership ``root`` and permissions ``0755`` to **ssh-ldap-pubkey** and **ssh-ldap-pubkey-wrapper**.

## Updating ssh-ldap-pubkey program

ssh-ldap-pubkey file use ldap.conf file so we need to give the ldap.conf file path in the ssh-ldap-pubkey file. The file path in your system might be different from ldap.conf file path in my system.

```shell
vim /usr/bin/ssh-ldap-pubkey

 DEFAULT_CONFIG_PATH = '/etc/openldap/ldap.conf'
```

Note Configure the ldap.conf according to your requirement.

After that try to run the ssh-ldap-pubkey manually to see does it fetch key or not. Also be sure that you have added the entry of a user with ssh-key in ldap database
Following is one way to run the command:

```shell
ssh-ldap-pubkey list -b ou=People,dc=example,dc=com -u username
```

## Configuring sshd_config file

```shell
vim  /etc/ssh/sshd_config 

PubkeyAuthentication yes

AuthorizedKeysCommand /usr/bin/ssh-ldap-pubkey-wrapper

AuthorizedKeysCommandUser root

PasswordAuthentication no

UsePAM yes
```

## Configuring sssd.conf file

```shell
vim /etc/sssd/sssd.conf

[domain/default]
id_provider = ldap
enumerate = true
auth_provider = ldap
ldap_uri = ldap://ldap.server.ip.or.dns
ldap_search_base = dc=example,dc=com
cache_credentials = True
ldap_tls_cacertdir = /etc/openldap/certs
ldap_tls_reqcert = allow
ldap_access_order = filter, expire
ldap_user_ssh_public_key = sshkey
ldap_use_tokengroups = False
sudo_provider = ldap

[sssd]
config_file_version = 2
services = nss, pam, sudo, ssh
domains = default
```

### Configuring /etc/pam.d/ssh file

```shell
vim /etc/pam.d/sshd

auth    sufficient      pam_ldap.so
account sufficient      pam_permit.so
```

### Configuring /etc/pam.d/common-session

```shell
vim /etc/pam.d/common-session

session     required      pam_mkhomedir.so skel=/etc/skel umask=0022
```

### Configuring /etc/pam.d/common-password

```shell
vi /etc/pam.d/common-password

password required pam_deny.so
```

### Configuring /etc/nsswitch.conf

```shell
vim /etc/nsswitch.conf

passwd: compat  sss
group:  compat  sss
shadow: compat  sss
```

### Restarting the Services

```shell
systemctl restart sssd nscd sshd
```

#### Now you can test the setup

For testing create a user on a localsystem then generate ssh keys, open the public key and copy the content of that key and add that content on the ldap user's database entry on ldap server. After that try running the ssh-ldap-pubkey program on the ldap client and see if it fetch the key, lets say it fetches then you can try to login from localsystem to the remote ldap client.
