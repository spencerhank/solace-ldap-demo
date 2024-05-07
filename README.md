## Open LDAP
Stands up openldap container for use with local Solace Software Broker.

* LDAP Configuration currently expects to be configured for use with a message vpn with name "default"
* Container has custom schema loaded to enable the "memberOf" attribute.
* Container has custom ldif loaded to mimic groups and client usernames created when using "Internal Database" as the authentication type.
* Note: If client username exists on the broker, then the broker defaults to "Internal Database" for authentication and authorization rather then the configured LDAP server.

Apache Directory Studio
* Used for viewing LDAP server contents
* Available at https://directory.apache.org/studio/downloads.html
* Connection Details
    * ```shell
    URL: ldap://localhost:1389
    Admin Dn: cn=admin,dc=example,dc=org
    Password: adminpassword
    ```
* Apple Silicon Users: Apache Directory Studio requires an x86_64 version of a jdk and point Apache Directory Studio to use this version.
    * Versions available at https://adoptium.net/temurin/releases/?arch=x64&version=21&os=mac
    * Update /Applications/ApacheDirectoryStudio.app/Contents/Info.plist to use the correct vm.
    * Example:
    ````
    <array>
            <!-- to use a specific Java version (instead of the platform's default) uncomment one of the following options,
                                            or add a VM found via $/usr/libexec/java_home -V
                                    <string>-vm</string><string>/System/Library/Java/JavaVirtualMachines/1.6.0.jdk/Contents/Commands/java</string>
                                    <string>-vm</string><string>/Library/Java/JavaVirtualMachines/1.8.0.jdk/Contents/Home/bin/java</string>
                            -->
            <string>-vm</string><string>/Library/Java/JavaVirtualMachines/temurin-20.jdk/Contents/Home/bin/java</string>
            <string>-keyring</string>
            <string>~/.eclipse_keyring</string>

        </array>
    ````


### Sample commands below for configuring LDAP.
* see ../semp.sh for implmentation

* configure ldap server for default profile
```shell
<rpc semp-version="soltr/10_6_1VMR">
    <authentication>
        <ldap-profile>
            <profile-name>default</profile-name>
            <ldap-server>
                <ldap-host>ldap://openldap:1389</ldap-host>
                <server-index>1</server-index>
            </ldap-server>
        </ldap-profile>
    </authentication>
</rpc>
```
* add admin creds
```shell
<rpc semp-version="soltr/10_6_1VMR">
    <authentication>
        <ldap-profile>
            <profile-name>default</profile-name>
            <admin>
                <admin-dn>cn=admin,dc=example,dc=org</admin-dn>
                <admin-password>adminpassword</admin-password>
            </admin>
        </ldap-profile>
    </authentication>
</rpc>
```
* configure base search and filter
```shell
<rpc semp-version="soltr/10_6_1VMR">
    <authentication>
        <ldap-profile>
            <profile-name>default</profile-name>
            <search>
                <base-dn>
                    <distinguished-name>ou=users,dc=example,dc=org</distinguished-name>
                </base-dn>
            </search>
        </ldap-profile>
    </authentication>
</rpc>
```
```shell
<rpc semp-version="soltr/10_6_1VMR">
    <authentication>
        <ldap-profile>
            <profile-name>default</profile-name>
            <search>
                <filter>
                    <filter>(&amp;(cn=$CLIENT_USERNAME)(memberOf=cn=$VPN_NAMEVpn,ou=groups,dc=example,dc=org))</filter>
                </filter>
            </search>
        </ldap-profile>
    </authentication>
</rpc>
```
* Configure authorization groups
```shell
curl --location 'https://localhost:1943/SEMP/v2/config/msgVpns/default/authorizationGroups' \
--header 'Content-Type: application/json' \
--header 'Accept: application/json' \
--data '{
    "aclProfileName": "acl-connector",
    "authorizationGroupName": "cn=client-connector,cn=defaultVpn,ou=groups,dc=example,dc=org",
    "clientProfileName": "client-connector",
    "enabled": true,
    "msgVpnName": "default"

}'
```
*  enable ldap-profile
```shell
<rpc semp-version="soltr/10_6_1VMR">
    <authentication>
        <ldap-profile>
            <profile-name>default</profile-name>
            <no>
                <shutdown></shutdown>
            </no>
        </ldap-profile>
    </authentication>
</rpc>
```
* Set as authentication and authorization type
```shell
curl --location --request PATCH 'https://localhost:1943/SEMP/v2/config/msgVpns/default' \
--header 'Content-Type: application/json' \
--header 'Accept: application/json' \
--data '{
        "authenticationBasicEnabled": true,
        "authenticationBasicProfileName": "default",
        "authenticationBasicType": "ldap",
        "authorizationLdapGroupMembershipAttributeName": "memberOf",
        "authorizationLdapTrimClientUsernameDomainEnabled": false,
        "authorizationProfileName": "default",
        "authorizationType": "ldap",
        "msgVpnName": "default"
    }'
```