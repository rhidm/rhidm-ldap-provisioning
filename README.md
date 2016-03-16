# Provisioning Users and Groups in Red Hat Identity Management via LDAP

## Preparing the IdM Server

### Create an account for the provisioning system and grant appropriate privileges

On the IdM server, become admin:
```
# kinit admin
```

Create a group (if one does not already exist) that will contain service/application accounts:
```
# ipa group-add service-accounts
```

Create a password policy, applied to this group of service-accounts, which prevents expiration and lockout DoS, but compensates with complex passwords
```
# ipa pwpolicy-add service-accounts --maxlife=10000 --minlife=0 --history=0 --minclasses=4 \
 --minlength=20 --priority=1 --maxfail=0 --failinterval=1 --lockouttime=0
```

Create a new account to be used by the provisioning system:
```
# ipa user-add provisionator --first=provisioning --last=account --password
```

Create a new account to be used to automatically activate all newly created users:
```
# ipa user-add activator --first=activation --last=account --password
```

Add the provisioning and activation accounts to the service account group, so that the password policy applies:
```
# ipa group-add-member service-accounts --users=provisionator
# ipa group-add-member service-accounts --users=activator
```

Grant the "User Administrator" role to the provisioning and activation accounts, which allows this account to read/write user and group information:
```
# ipa role-add-member --users=provisionator "User Administrator"
# ipa role-add-member --users=activator "User Administrator"
```

As all new user accounts are created with an immediately expiring password, change the accounts' passwords now.:
```
# kpasswd provisionator
# kpasswd activator
```

### Enable process for automatically activating newly created user accounts
**Perform the following steps on *at least one* of the IdM servers.**

Generate a keytab file for the activation account.  Note that if this process will run on more than one IdM server, the keytab generation must be run on only the first, and the keytab file then copied to the others.
```
ipa-getkeytab -s ipa.example.com -p "activator" -k /etc/krb5.ipa-activation.keytab
```

Create the executable script */usr/local/sbin/ipa-activate-all*:
```
#!/bin/bash

kinit -k -i activator

ipa stageuser-find --all --raw | grep "  uid:" | cut -d ":" -f 2 | while read uid; do ipa stageuser-activate ${uid}; done

```

```
# chmod 755 /usr/local/sbin/ipa-activate-all
# chown root.root /usr/local/sbin/ipa-activate-all 
```

Create the systemd unit file */etc/systemd/system/ipa-activate-all.service*:

```
[Unit]
Description=Scan IPA every minute for any staged users that need to be activated

[Service]
Environment=KRB5_CLIENT_KTNAME=/etc/krb5.ipa-activation.keytab
Environment=KRB5CCNAME=FILE:/tmp/krb5cc_ipa-activate-all
ExecStart=/usr/local/sbin/ipa-activate-all
```

Create a systemd timer */etc/systemd/system/ipa-activate-all.timer*:

```
[Unit]
Description=Scan IPA every minute for any staged users that need to be activated

[Timer]
OnBootSec=15min
OnUnitActiveSec=1min

[Install]
WantedBy=multi-user.target
```

```
# systemctl enable ipa-activate-all.timer
```

## Provisioning Users
### Add new user
Note that all new users are created as "Staged Users" - simple LDAP objects that do not yet have additional IPA-specific attributes.  Once users are staged, it is necessary to "activate" them, which associates the additional IP attributes.

The following LDIF-formatted templates should provide a basic framework for configuring the LDAP provider of a provisioning system.  For manual testing, these can also be used with the ldapmodify command:
```
ldapmodify  -x -D "uid=provisionator,cn=users,cn=accounts,dc=example, dc=com" \
 -w "$PROVPW" -H ldap://ipa.example.com
```

Create a new user with GID and UID automatically assigned:
```
dn: uid=$USERID,cn=staged users, cn=accounts, cn=provisioning, dc=example, dc=com
changetype: add
objectClass: top
objectClass: inetorgperson
uid: $USERID
sn: $SN
givenName: $GIVENNAME
cn: $CN
```

Create a new user with GID and UID statically assigned:
```
dn: uid=$USERID,cn=staged users, cn=accounts, cn=provisioning, dc=example, dc=com
changetype: add
objectClass: top
objectClass: person
objectClass: inetorgperson
objectClass: organizationalperson
objectClass: posixaccount
uid: $USERID
uidNumber: $UIDNUMBER
gidNumber: $GIDNUMBER
sn: $SN
givenName: $GIVENNAME
cn: $CN
homeDirectory: /home/$USERID
```

## Modifying existing users
In all cases, the DN of the user first needs to be obtained by searching by user id:
```
# ldapsearch -LLL -x -D "uid=provisionator,cn=users,cn=accounts,dc=example, dc=com" \
 -w "$PROVPW" -H ldap://ipa.example.com -b "cn=users, cn=accounts, dc=example, dc=com" uid=$USERID 1.1
```

Note that all users in IPA are in the "accounts" tree, so all user DNs will be of the form 
**uid=$USERID,cn=users,cn=accounts,dc=example, dc=com**.  But it is still recommended practice to search by uid to find the correct DN.

The following LDIF samples can be used as templates for a provisioning system, or tested with the ldapmodify command using:
```
# ldapmodify  -x -D "uid=provisionator,cn=users,cn=accounts,dc=example, dc=com" \
 -w "$PROVPW" -H ldap://ipa.example.com
```

### Modify attributes of user
```
dn: $DN
changetype: modify
replace: $ATTRIBUTE
$ATTRIBUTE: $VALUE
```

### Disable user
```
dn: $DN
changetype: modify
replace: nsAccountLock
nsAccountLock: TRUE
```

### Re-enable user
```
dn: $DN
changetype: modify
replace: nsAccountLock
nsAccountLock: FALSE
```

### Archive user
```
dn: $DN
changetype: modrdn
newrdn: uid=$USERID
deleteoldrdn: 0
newsuperior: cn=deleted users,cn=accounts,cn=provisioning,dc=example
```

## Modifying groups
In all cases, the DN of the group needs to be obtained by searching by group name:
```
# ldapsearch -YGSSAPI  -H ldap://ipa.example.com -b "cn=groups, cn=accounts, dc=example, dc=com" "cn=$GROUPNAME"
```

Note that all groups in IPA are in the "accounts" tree, so all user DNs will be of the form 
**uid=$GROUPNAME,cn=groups,cn=accounts,dc=example, dc=com**.  But it is still recommended practice to search by cn to find the correct DN.

The following LDIF samples can be used as templates for a provisioning system, or tested with the ldapmodify command using:
```
# ldapmodify  -x -D "uid=provisionator,cn=users,cn=accounts,dc=example, dc=com" \
 -w "$PROVPW" -H ldap://ipa.example.com
```

### Create a new group
```
dn: cn=$GROUPNAME,cn=groups,cn=accounts,dc=example, dc=com
changetype: add
objectClass: top
objectClass: ipaobject
objectClass: ipausergroup
objectClass: groupofnames
objectClass: nestedgroup
objectClass: posixgroup
cn: $GROUPNAME
gidNumber: $GIDNUMBER
```

### Delete an existing group
```
dn: $DN
changetype: delete
```

### Add a member to a group
```
dn: $DN
changetype: modify
add: member
member: uid=$USERID,cn=users,cn=accounts,dc=example, dc=com
```

### Remove a member from a group
```
dn: $DN
changetype: modify
delete: member
member: uid=$USERID,cn=users,cn=accounts,dc=example, dc=com
```
