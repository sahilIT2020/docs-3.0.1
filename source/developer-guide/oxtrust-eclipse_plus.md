# How to Build oxTrust with Eclipse

Sections till LDAP installation from oxtrust-eclipse.md

## Install and configure Symas OpenLDAP

1\. Download Silver Edition from: https://downloads.symas.com/SDLPWeb

2\. Create folder for custom Gluu schema: "C:\Program Files (x86)\symas-openldap\etc\openldap\schema"

3\. Copy into custom Gluu schema folder 2 files from CE /opt/gluu-server-3.0.1/opt/gluu/schema/openldap

4\. Copy "C:\Program Files (x86)\symas-openldap\etc\openldap\slapd.conf.default" into "C:\Program Files (x86)\symas-openldap\etc\openldap\slapd.conf"

5\. Edit file "C:\Program Files (x86)\symas-openldap\etc\openldap\slapd.conf"
 - Uncommnet next lines:
```
   include		"etc/openldap/schema/ppolicy.schema"
   include		"etc/openldap/schema/cosine.schema"
   include		"etc/openldap/schema/inetorgperson.schema"
   include		"etc/openldap/schema/eduperson.schema"
```
 - Add next include lines:
```
   include		"etc/openldap/gluu/gluu.schema"
   include		"etc/openldap/gluu/custom.schema"
```

 - Uncomment modules:
```
   moduleload	ppolicy.la
   moduleload	unique.la
```

 - Copy from CE file /opt/gluu-server-3.0.1/opt/symas/etc/openldap/slapd.conf sections into "C:\Program Files (x86)\symas-openldap\etc\openldap\slapd.conf":
```
   #######################################################################
   # Main Database housing all the o=gluu info
   #######################################################################
   ...
   #######################################################################
   # Site database housing o=site information
   #######################################################################
```
   Hint:
   End last section is after line:
   index	gluuStatus


  - Replace in sections "Main Database" and "Site database":
     1. "database	mdb" with "database	hdb"
     2. rootpw with your clear text password.
     3. directory location "/opt/gluu/data" with "var/openldap-data".
  - Remove in sections "Main Database" and "Site database" maxsize option.

6\. Create new DB folders:
  - "C:\Program Files (x86)\symas-openldap\var\openldap-data\main_db"
  - "C:\Program Files (x86)\symas-openldap\var\openldap-data\site_db"

7\. Copy default DB settings (rename DB_CONFIG.default to DB_CONFIG during copy):
  - "C:\Program Files (x86)\symas-openldap\etc\openldap\DB_CONFIG.default" into "C:\Program Files (x86)\symas-openldap\var\openldap-data\main_db\DB_CONFIG"
  - "C:\Program Files (x86)\symas-openldap\etc\openldap\DB_CONFIG.default" into "C:\Program Files (x86)\symas-openldap\var\openldap-data\site_db\DB_CONFIG"

8\. Verify OpenLDAP settings:
```
   slaptest.bat -f "C:\Program Files (x86)\symas-openldap\etc\openldap\slapd.conf"
   ...
   config file testing succeeded
```

9\. Now we can try to run OpenLDAP service and connect to LDAP server localhost:389

## Import data from CE into dev LDAP

1\. Export "o=gluu" tree in CE into gluu.ldif
```
export OPENDJ_JAVA_HOME=/opt/jre; /opt/opendj/bin/ldapsearch -h localhost -p 1636  -Z -X -w secret -D "cn=directory manager,o=gluu" -b "o=gluu" objectClass=* > gluu.ldif
```

2\. Load gluu.ldif into dev LDAP and update to conform new environemt

3\. All Gluu applciations store setting in LDAP. Hence we need to update their configuration in LDAP

3.1 We need to change authentication setting: inum=<appliance_inum>,ou=appliances,o=gluu. We need to remove IDPAuthentication attribute from this entry.

3.2 Fix invalid cache setting JSON format in: inum=<appliance_inum>,ou=appliances,o=gluu. We need to remove do:
  - Replace IN_MEMORY with "IN_MEMORY"
  - Replace DEFAULT with "DEFAULT"

3.3 We need to change oxAuth settings: ou=oxauth,ou=configuration,inum=<appliance_inum>,ou=appliances,o=gluu. We need to apply next changes to oxAuthConfDynamic attribute value.
  - Replace "https://<ce_host_name>/oxauth" with "https://localhost:8443/oxauth"
  - Replace  "issuer":"https://<ce_host_name>" with "oxAuthIssuer":"https://localhost:8443"

3.4 We need to change oxTrust settings: ou=oxtrust,ou=configuration,inum=<appliance_inum>,ou=appliances,o=gluu. We need to apply next changes to oxTrustConfApplication attribute value.
  - Replace "https://<ce_host_name>/identity" with "https://localhost:8453/identity"
  - Replace "https://<ce_host_name>/oxauth" with "https://localhost:8443/oxauth"
  - Replace  "oxAuthIssuer":"https://<ce_host_name>" with "oxAuthIssuer":"https://localhost:8443/oxauth"
  - Replace  "umaIssuer":"https://<ce_host_name>" with "umaIssuer":"https://localhost:8443/oxauth"

3.5 Fix oxTrust oxAuth client settings: inum=<org_inum>!0008!8CF0.83A5,ou=clients,o=<org_inum>,o=gluu. We need to add next attribute values:
  - oxAuthRedirectURI: https://localhost:8453/identity/authentication/authcode
  - oxAuthPostLogoutRedirectURI: https://localhost:8453/identity/authentication/finishlogout

## Start oxAuth under Jetty in Eclipse
