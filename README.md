LDAP installation.

1. sudo su - 
2. install open ldap
```shell
 yum -y install openldap compat-openldap openldap-clients openldap-servers openldap-servers-sql openldap-devel
 ```
3. Start the ldap service
```shell
 systemctl start slapd

 systemctl enable slapd
```
4. generate a passwd
```shell
 slappasswd -h {SSHA} -s admin1 #(this is the passwd for ldap admin, replae with your own password)
 ```
 The above command will return a encrypted password, note it down, you will need in the next steps.

5. create a file configuring the ldap server access.

```shell
cat <<EOT >> db.ldif
dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=hortonworks,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=ldapadm,dc=hortonworks,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}6c31I7JVxJaiK2zGNspxm+IAEHPie9If
EOT
```
6. open the dp.ldif file and change olcRootPW value to the password generated in step 4

change the dc=hortonworks ,dc=com to match you domain.

ldapadm is the bind dn or the admin user for this ldap. change it to a name of choice if needed

7. push the configuration to the ldap server
```shell
ldapmodify -Y EXTERNAL  -H ldapi:/// -f db.ldif
```

8.restrict monitor access to just ldapadm user
```shell
cat <<EOT >> monitor.ldif
dn: olcDatabase={1}monitor,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external, cn=auth" read by dn.base="cn=ldapadm,dc=hortonworks,dc=com" read by * none
EOT
```
9. push configuration to ldap servie
```shell
ldapmodify -Y EXTERNAL  -H ldapi:/// -f monitor.ldif
```

 10. run the following commands to setup the ldap DB.
 ```shell
 cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
 chown ldap:ldap /var/lib/ldap/*

 ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
 ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif 
 ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
 ```
 11. Generate the base users for ldap.

 ```shell
 cat <<EOT >> base.ldif
 dn: dc=hortonworks,dc=com
dc: hortonworks
objectClass: top
objectClass: domain

dn: cn=ldapadm ,dc=hortonworks,dc=com
objectClass: organizationalRole
cn: ldapadm
description: LDAP Manager

dn: ou=People,dc=hortonworks,dc=com
objectClass: organizationalUnit
ou: People

dn: ou=Group,dc=hortonworks,dc=com
objectClass: organizationalUnit
ou: Group
EOT
 ```

 12. create the LDAP tree structure
 ```shell
 ldapadd -x -W -D "cn=ldapadm,dc=hortonworks,dc=com" -f base.ldif
 ```
 You will be prompted for LDAP password, this is the password that your provide to the slappasswd command in step 4. give the text password. in my case it was "admin1"

 13. Add a few users
 ```shell
 cat <<EOT >> user1.ldif
 dn: uid=user1,ou=people,dc=hortonworks,dc=com
 objectclass:top
 objectclass:person
 objectclass:organizationalPerson
 objectclass:inetOrgPerson
 cn: User1
 sn: User
 uid: user1
 userPassword: admin1
 EOT
 ```
 You will be prompted for ldap password ("admin1" in my case)
 ```shell
 cat <<EOT >> dpprofiler.ldif
 dn: uid=dpprofiler,ou=people,dc=hortonworks,dc=com
 objectclass:top
 objectclass:person
 objectclass:organizationalPerson
 objectclass:inetOrgPerson
 cn: Dpprofiler
 sn: User
 uid: dpprofiler
 userPassword: admin1
 EOT
 ```
 add users to ldap
 ```shell
 ldapadd -x -W -D "cn=ldapadm,dc=hortonworks,dc=com" -f user1.ldif
 ldapadd -x -W -D "cn=ldapadm,dc=hortonworks,dc=com" -f dpprofiler.ldif
 ```
 14. check if the users are there by doing a search
 ```shell
 ldapsearch -H ldap://dpsrepo0.field.hortonworks.com -b "dc=hortonworks,dc=com" -s sub "objectclass=inetOrdPerson" -x
 ```


This is the article i referred to do the install https://www.itzgeek.com/how-tos/linux/centos-how-tos/step-step-openldap-server-configuration-centos-7-rhel-7.html/2
