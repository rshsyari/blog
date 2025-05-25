---
date: 2025-03-31T23:55:07+07:00
# description: ""
# image: ""
# lastmod: 2025-03-31
showTableOfContents: true
# tags: ["",]
title: "LDAP Script"
type: "post"
---
Managing LDAP can sometimes get complicated. Especially if it gets to the point where there are a huge amount of users. Using shell script can help reduce the stress and increase efficiency in managing those users.

For example, here we have 2 shell scripts, one for adding users, and the other for deleting users. Let's take a closer look:

## Add User
```sh
#!/usr/bin/env bash

# Validate arguments
if [ -z $1 ] || [ -z $2 ]; then
	echo "Please run it as: ./add_ldap_user.sh <OU|none> <USER>"
	exit 1
fi

# Validate user; if exists, abort
if ldapsearch -xb dc=example,dc=com "uid=$2" | grep -q "uid: $2"; then
	echo "User $2 already exist!"
	exit 1
fi

# Validate OU; if none, omit; if invalid, abort
ou_check=$(echo $(ldapsearch -xb dc=example,dc=com "ou=$1" | grep -q "ou: $1") $?)

if [[ $ou_check == "0" ]]; then
	DN="cn=$2,ou=$1,dc=example,dc=com"

elif [[ ${1,,} == "none" ]]; then
	DN="cn=$2,dc=example,dc=com"

else
	echo "Organizational Unit $1 is invalid, aborting..."
fi

# Set encrypted password
pass=$(slappasswd -s very_secure_password)

# Execute user addition
ldapadd -xD cn=admin,dc=example,dc=com -w very_secure_password <<EOF
dn: $DN
objectClass: top
objectClass: inetOrgPerson
objectClass: posixAccount
cn: $2
sn: $2
uid: $2
gecos: $2
userPassword: $pass
uidNumber: $((1100 + $RANDOM % 10000))
gidNumber: $((1100 + $RANDOM % 10000))
homeDirectory: /home/$2
loginShell: /bin/bash
description: $2 account
EOF
```

With scripts like this, you no longer have to write manual ldif file for each user, or spending a lot of times just to add some users. You can just write the bash script in the intended way, and poof! The user exists...

If you are using LDAP over SSL or LDAP with STARTTLS, I suggest you take a closer look and modify some parts of the script yourself, because I used plain LDAP connection when writing the script.

Anyway, here is one for user deletion..

## Delete User
```sh
#!/usr/bin/env bash

# Validate arguments
if [ -z $1 ]; then
	echo "Please run it as: ./delete_ldap_user.sh <USER_TO_DELETE>"
	exit 1
fi

if ! ldapsearch -xb dc=example,dc=com "cn=$1" | grep -q "cn=$1"; then
	echo "User $1 doesn't exists on LDAP database"
	exit 1
fi

# Define user DN
DN=$(ldapsearch -x -LLL -b "dc=example,dc=com" "(uid=$1)" | grep ^dn | awk '{ print $NF }'

# Executes the deletion
ldapdelete -xD cn=admin,dc=example,dc=com -w very_secure_password $DN

if [ $? -eq 0 ]; then
	echo "User $1 has been deleted"
else
	echo "User $1 deletion failed"
	exit 1
fi
```

Or if you want to, you could specify the OU in the deletion script, so it would be run as: `bash delete_ldap_user.sh <OU> <USER>` just like the addition script.

But let's say for example, you want to add LDAP users by converting it from a CSV File, well for this kinda task, I myself prefer to use Ansible! Let's see some example below...

## Import from CSV
```yaml
- name: Configure LDAP server
  hosts: ldap
  gather_facts: false
  no_log: yes
  tasks:
  - name: Read CSV File
    read_csv:
      path: /etc/ansible/users.csv
      delimiter: ';'
    register: users
    delegate_to: localhost

  - name: Make sure slapd is installed
    apt:
      name: slapd

  - name: Import users to ldap
    community.general.ldap_entry:
      state: present
      server_uri: ldap://localhost/
      bind_dn: cn=admin,dc=example,dc=com
      bind_pw: very_secure_password
      dn: uid="{{ item.username }}",dc=example,dc=com
      objectClass:
        - top
        - inetOrgPerson
        - posixAccount
      attributes:
        uid: "{{item.username}}"
        cn: "{{ item.username }}"
        uidNumber: "{{ item.uid }}"
        gidNumber: "{{ item.uid }}"
        homeDirectory: "{{ item.home }}"
        loginShell: /bin/bash
        userPassword: "{{ item.password }}"
    loop: "{{ users.list }}"
```
