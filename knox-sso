1. install ambari 2.6.2.2
2. install hdp 2.6.5 (ensure hive,spark2,livy2,atlas,ranger,knox are installed)
3. We will setup knox sso to work with LDAP. you can use existing LDAP or install a new LDAP. you can use these instructions for installing your own ldp https://github.com/hortonworks/sedev/blob/master/dps_dss_install_ldap/install_ldap.md

To setup knox SSo copy the followng xml in to knox configs under Ambari > Knox > configs > Advanced Knoxsso topology, replace the ldap url , binddn, bindpassword and search base to match your ldap install. 
```xml
<topology>
<gateway>
<provider>
<role>authentication</role>
<name>ShiroProvider</name>
<enabled>true</enabled>
<param name="main.ldapRealm" value="org.apache.hadoop.gateway.shirorealm.KnoxLdapRealm"/>
<param name="main.ldapContextFactory" value="org.apache.hadoop.gateway.shirorealm.KnoxLdapContextFactory"/>
<param name="main.ldapRealm.contextFactory" value="$ldapContextFactory"/>
<param name="main.ldapRealm.contextFactory.url" value="ldap://<ldap_host>:389"/>
<param name="main.ldapRealm.contextFactory.systemUsername" value="cn=ldapadm,dc=hortonworks,dc=com"/>
<param name="main.ldapRealm.contextFactory.systemPassword" value="admin1"/>
<param name="main.ldapRealm.searchBase" value="dc=hortonworks,dc=com"/>
<param name="main.ldapRealm.userSearchAttributeName" value="uid"/>
<param name="main.ldapRealm.userObjectClass" value="inetOrgPerson"/>
<param name="urls./**" value="authcBasic"/>
</provider>
</gateway>
<application>
<name>knoxauth</name>
</application>
<service>
<role>KNOXSSO</role>
<param>
  <name>knoxsso.cookie.secure.only</name>
  <value>false</value>
</param>
<param>
  <name>knoxsso.token.ttl</name>
  <value>30000</value>
</param>
<param>
 <name>knoxsso.redirect.whitelist.regex</name>
 <value>^https?:\/\/(\.hortonworks\.com|.*\.hortonworks\.com|localhost|127\.0\.0\.1|0:0:0:0:0:0:0:1|::1):[0-9].*$</value>
</param>
</service>
</topology>
```
Change the knoxsso.redirect.whitelist.regex to match your domain. save the config.

We will now update the token.xml this will allow ambari,ranger and atlas to use knoxsso for authentication.

grab the token.xml below and modify sso.authentication.provider.url, to point to your knox host and port.

we also modify sso.token.verification.pem , to the key from your knox installation. To get the key, ssh in to you knox_host and run the command below...
```shell
echo | openssl s_client -connect `hostname -f`:8443 2>&1 | sed --quiet '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' | grep -v CERTIFICATE
```
copy the key output by the above command and paste in the below xml template. keep this token somewhere accessible you will need it when enabling sso in ambari,ranger and atlas.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<topology>
<uri>https://[replace_knox_host]:8443/gateway/token</uri>
<name>token</name>
<gateway>
<provider>
<role>federation</role>
<name>SSOCookieProvider</name>
<enabled>true</enabled>
<param>
<name>sso.authentication.provider.url</name>
<value>https://[replace-knox-host]:8443/gateway/knoxsso/api/v1/websso</value>
</param>
<param>
<name>sso.token.verification.pem</name>
<value>
<!--<token goes here>-->
</value>
</param>
</provider>
<provider>
<role>identity-assertion</role>
<name>HadoopGroupProvider</name>
<enabled>true</enabled>
</provider>
<provider>
<role>authorization</role>
<name>XASecurePDPKnox</name>
<enabled>true</enabled>
</provider>
</gateway>
<service>
<role>KNOXTOKEN</role>
<param>
<name>knox.token.ttl</name>
<value>500000</value>
</param>
<param>
<name>knox.token.client.data</name>
<value>cookie.name=hadoop-jwt</value>
</param>
<param>
<name>main.ldapRealm.authorizationEnabled</name>
<value>true</value>
</param>
</service>
</topology>

```

ssh to the knox_host. 
```shell
cd /etc/knox/conf/topologies
vi token.xml
#paste the modified token xml and save.
```

on ambari restart knox

4. Enable ambari to use LDAP, run the commands below on ambari-host.
```shell
#put ldap admin password in a data file for ambari
echo "admin1" > /etc/ambari-server/conf/ldap-password.dat
#back ambari properties
cp /etc/ambari-server/conf/ambari.properties /etc/ambari-server/conf/ambari.properties.back

###copy all the lines below as one and execute on shell, modify the dc= , to match your domain, modify LDAP_SERVER to ldap host
echo "ambari.ldap.isConfigured=true
authentication.ldap.baseDn=dc=hortonworks,dc=com
authentication.ldap.bindAnonymously=false
authentication.ldap.dnAttribute=dc=hortonworks,dc=com
authentication.ldap.groupMembershipAttr=member
authentication.ldap.groupNamingAttr=cn
authentication.ldap.groupObjectClass=groupOfNames
authentication.ldap.managerDn=cn=ldapadm,dc=hortonworks,dc=com
authentication.ldap.managerPassword=/etc/ambari-server/conf/ldap-password.dat
authentication.ldap.primaryUrl=<LDAP_SERVER>:389
authentication.ldap.referral=ignore
authentication.ldap.useSSL=false
authentication.ldap.userObjectClass=inetOrgPerson
authentication.ldap.usernameAttribute=uid
client.security=ldap
ldap.sync.username.collision.behavior=convert" >> /etc/ambari-server/conf/ambari.properties

### till here is one command

# restart ambari server
ambari-server restart

#sync ldap users to ambari
ambari-server sync-ldap --all 
```
Now login back to ambari and make sure you select one of the ldap users as admin. once SSO is enabled you wont be able to login as the local user. to do that login to ambari and got admin > manage ambari

<img src="imgs/manage.png" alt="manage ambari" width="400" height="250">

select the user from users list and toggle admin option

<img src="imgs/users.png" alt="manage ambari" width="400" height="250">


Now go back to ambari-host terminal and execute the following command.

```shell
amabri-server setup-sso
```
select y on prompt to enable sso
enter the knox sso url when prompted for provider url : https://[replace-knox-host]:8443/gateway/knoxsso/api/v1/websso
Paste the certifcate that you had copied from know earlier in the document when prompted for certificate.
choose defaults for the remaining prompts.

restart ambari
```shell
ambari-server restart
```
go to ambari ui , you should get the knox login screen

<img src="imgs/knox.png" alt="manage ambari" width="400" height="250">

5. Enable Ranger to use LDAP. 
Under ambari > Ranger > Configs > Ranger User Info update as follows 

set Sync Source - LDAP/AD

under common configs

LDAP/AD URL : ldap://[YOUR_LDAP_HOST]:389
Bind User : cn=ldapadm,dc=hortonworks,dc=com # change per your domain if your admin and domain is different.
Bind password : admin1 # password for bind user

under user configs tab

username attribute : uid
User Object Class : inteOrgPerson # change per your ldap
User Search Base:dc=hortonworks,dc=com # change per your domain

under group configs tab

if you deployed you own ldap from this article, then you can turn off the group sync. if you deployed your own, you could the settings as below

Group Member Attribute : member
Group Name Attribute : cn
Group Object Class : groupOfNames
Group Search Base : ou=groups,dc=hortonworks,dc=com

save and restart ranger. Login to ranger admin ui and make a ldap user as admin.

6. enable Ranger SSO
The key copied is the same key that we go from knox,earlier in the document
<img src="imgs/ambari-sso.png" alt="manage ambari" width="400" height="250">

restart ranger. after restart click on quicklinks > ranger admin ui . you should be logged into ranger with same user as ambari.

7. Enable ldap for Atlas (please do this after installing dp profiler )

on Ambari go to Atlas > configs and update as follows. the values for bind, base search etc for ldap will be different if you used a different ldap and did not install one based on this doc.

uncheck file authentication

check LDAP Authentication

LDAP Authentication Type :- LDAP
  
atlas.authentication.method.ldap.url :- ldap://<ldap_host>:389

atlas.authentication.method.ldap.userDNpattern :- uid={0],ou=people,dc=hortonworks,dc=com

atlas.authentication.method.ldap.groupSearchBase :- dc=hortonworks,dc=com

atlas.authentication.method.ldap.groupSearchFilter :- atlas.authentication.method.ldap.groupRoleAttribute :- cn

atlas.authentication.method.ldap.base.dn :-dc=hortonworks,dc=com

atlas.authentication.method.ldap.bind.dn :- dc=ldapadm,dc=hortonworks,dc=com

atlas.authentication.method.ldap.bind.password :- admin1 or the password for admin user

atlas.authentication.method.ldap.referral :- ignore

atlas.authentication.method.ldap.user.searchfilter :- uid


8. Enable Atlas SSO (please do this after installing dp profiler )

Enable SSO by checking Enable Atlas Knox SSo.

Copy the certificate that we have saved from knox in atlas.sso.knox.publicKey.

Restart Atlas.

Now Ranger,Atlas,Ambari all are using knox sso.

9. perform the pre-requisite steps from configure dp-proxy in knox

cd /etc/knox/conf/topologies and open a file dp-proxy.xml. paste below xml content,change localhost to the corresponding service hostname.
```xml

<topology>
       
  <gateway>
             
    <provider>
                   
      <role>federation</role>
                   
      <name>SSOCookieProvider</name>
                   
      <enabled>true</enabled>
                   
      <param>
                         
        <name>sso.authentication.provider.url</name>
                         
        <value>https://localhost:8443/gateway/knoxsso/knoxauth/login.html</value>
                     
      </param>
               
    </provider>
             
    <provider>
                   
      <role>identity-assertion</role>
                   
      <name>Default</name>
                   
      <enabled>true</enabled>
               
    </provider>
         
  </gateway>
       
  <service>
             
    <role>WEBHDFS</role>
             
    <url>http://ctr-e138-1518143905142-369209-01-000005.hwx.site:20070</url>
         
  </service>
       
  <service>
             
    <role>WEBHCAT</role>
             
    <url>http://ctr-e138-1518143905142-369209-01-000005.hwx.site:20111/templeton</url>
         
  </service>
       
  <service>
             
    <role>AMBARI</role>
             
    <url>http://ctr-e138-1518143905142-369209-01-000006.hwx.site:8080</url>
         
  </service>
       
  <service>
             
    <role>RANGER</role>
             
    <url>http://ctr-e138-1518143905142-369209-01-000005.hwx.site:6080</url>
         
  </service>
       
  <service>
             
    <role>RANGERUI</role>
             
    <url>http://ctr-e138-1518143905142-369209-01-000005.hwx.site:6080</url>
         
  </service>
       
  <service>
             
    <role>ATLAS</role>
             
    <url>http://ctr-e138-1518143905142-369209-01-000005.hwx.site:21000</url>
         
  </service>
       
  <service>
             
    <role>ATLAS-API</role>
             
    <url>http://ctr-e138-1518143905142-369209-01-000005.hwx.site:21000</url>
         
  </service>
       
  <service>
             
    <role>OOZIE</role>
             
    <url>none</url>
         
  </service>
       
  <service>
             
    <role>WEBHBASE</role>
             
    <url>http://ctr-e138-1518143905142-369209-01-000005.hwx.site:30010</url>
         
  </service>
       
  <service>
             
    <role>HIVE</role>
             
    <url>http://ctr-e138-1518143905142-369209-01-000005.hwx.site:10001/cliservice</url>
         
  </service>
       
  <service>
             
    <role>RESOURCEMANAGER</role>
             
    <url>http://ctr-e138-1518143905142-369209-01-000005.hwx.site:8088/ws</url>
         
  </service>
       
  <service>
             
    <role>BEACON</role>
             
    <url>none</url>
         
  </service>
       
  <service>
             
    <role>PROFILER-AGENT</role>
             
    <url>http://ctr-e138-1518143905142-369209-01-000005.hwx.site:21900</url>
         
  </service>
   
</topology>

```
9. Install DSS profiler agent

DSS profiler agent requires postgres 9.3+ if using postgres. you can alternatley use mysql or an internal h2 database.

execute below commands to install postgres 9.6
``` shell
yum install https://download.postgresql.org/pub/repos/yum/9.6/redhat/rhel-7-x86_64/pgdg-redhat96-9.6-3.noarch.rpm

yum install postgresql96-server

#initialize database
/usr/pgsql-9.6/bin/postgresql96-setup initdb

 systemctl enable postgresql-9.6.service
 systemctl start postgresql-9.6.service
 
 #create a database for the profiler using the commands below
 
 sudo su - postgres
 
 psql
 
 # on postgres sql prompt
 
 create database profileragent;
 
 create user profileragent with password 'profiler';
 
 grant all privileges on database profileragent to profileragent;
 
 \q
 
```
To allow postgres to accept remote connections follow the steps in this link..

https://docs.hortonworks.com/HDPDocuments/HDF3/HDF-3.1.0/bk_installing-hdf-and-hdp/content/postgres-remote-connections.html

Now we will install the service in ambari

download the mpack from the DSS release page link.

install the mpack in ambari using the following commands

```shell

sudo ambari-server install-mpack --mpack= /path/to/mpack
sudo ambari-server restart
```
When ambari server restarts, login to the ambari ui to add the dss profier service. There is an issue with the mpack, where it does not provide the right url for the dss bits. It expects a local repo to have been created and points to localhost. To change that login to ambari, select the manager ambari menu from the user dropdown on the top right corner.

<img src="imgs/dss1.png" alt="manage ambari" width="400" height="250">

select versions and then select the version of hdp, hdp-2.6.5.0 in this case.

<img src="imgs/dss2.png" alt="manage ambari" width="400" height="250">

Change the base url for DSS to point to the baseurl in the repo file download from hwx or to a local repository wth dss bits.

<img src="imgs/dss3.png" alt="manage ambari" width="400" height="250">

Save the changes.

Now Click on Actions and , the select "Add Service " in ambari. From the list , select Data Proflier. Hit next and follow the prompts. Enter the details of the database, username and password that we setup earlier in the section. Select db type as postgres. Enter any password for the SPNEGO secret under the advanced section. Since we are not enabling kerberos, it will not have effect.

Follow the wizard and hit deploy at the end. Ambari should install and start the agent.

Some gotchas...

1. in my install the agent would fail complaining about no java installation found. in that case, open "sudo vi /usr/dss/current/profiler_agent/bin/profiler-agent" and add an export JAVA_HOME= to the top of that script.
2. in my install the profiler also complained about auth failure with atlas. disabling SSO and enabling file based authentication fixed it. looks like it needs user admin to create the profiler types
