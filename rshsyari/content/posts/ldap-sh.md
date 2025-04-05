---
date: 2025-03-31T23:55:07+07:00
# description: ""
# image: ""
# lastmod: 2025-03-31
showTableOfContents: false
# tags: ["",]
title: "LDAP Script"
type: "post"
---
Managing LDAP can sometimes get complicated. Especially if it gets to the point where there are a huge amount of users. Using shell script can help reduce the stress and increase efficiency in managing those users.

For example, here we have 2 shell scripts, one for adding users, and the other for deleting users. Let's take a closer look:

## Add User
```bash
#!/usr/bin/env bash

# Validate arguments
if [ -z $1 ] || [ -z $2 ]; then
	echo "Please run it as: ./add_ldap_user.sh <OU> <USER>"
	exit 1
fi

# Check if the user already exists
if ldapsearch -xb dc=example,dc=com "uid=$2" | grep -q "uid: $2"; then
	echo "User $2 already exist!"
	exit 1
fi

# Validate the OU; it should exist, if not then create it
if ! ldapsearch -xb dc=example,dc=com "ou=$1" | grep -q "ou: $1"; then
    echo "Organizational Unit '$1' does not exist! Creating..."
    ldapadd -xD cn=admin,dc=example,dc=com -w P@ssw0rd <<EOF
    dn: ou=$1,dc=example,dc=com
    objectClass: top
    objectClass: organizationalUnit
    ou: $1
    description: $1 organizational unit
EOF

# Execute the user addition
ldapadd -cxD cn=admin,dc=example,dc=com -w P@ssw0rd <<EOF
dn: uid=$2,ou=$1,dc=example,dc=com
objectClass: top
objectClass: inetOrgPerson
objectClass: posixAccount
cn: $2
sn $2
uid: $2
userPassword: $pass
uid: $((1100 + $RANDOM % 10000))
gid: $((1100 + $RANDOM % 10000))
homeDirectory: /home/$2
gecos: $2
description: $2 user account
EOF

if [ $? -eq 0 ]; then
    echo "User '$2' successfully added to OU '$1'."
else
    echo "Failed to add user '$2' to OU '$1'."
    exit 1
fi
```

With scripts like this, you no longer have to write manual ldif file for each user, or spending a lot of times just to add some users. You can just write the bash script in the intended way, and poof! The user exists...

## Delete User
```bash
#!/usr/bin/env bash

# Validate arguments
if [ -z $1 ]; then
	echo "Please run it as: ./delete_ldap_user.sh <USER_TO_DELETE>"
	exit 1
fi

# Validate user; it should exists in order to be deleted
if ! ldapsearch -xb dc=example,dc=com "uid=$2" | grep -q "uid: $2"
	echo "User $1 doesn't exist on LDAP database!"
	exit 1
fi

# Executes the deletion
ldapdelete -xD cn=admin,dc=example,dc=com -w P@ssw0rd "uid=$1,ou=*,dc=example,dc=com"

if [ $? -eq 0 ]; then
    echo "User $1 has been deleted!"
else
    echo "Failed to delete user '$1'."
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
      name: slapd,python3-ldap

  - name: Import users to ldap
    community.general.ldap_entry:
      state: present
      server_uri: ldap://localhost/
      bind_dn: cn=admin,dc=example,dc=com
      bind_pw: P@ssw0rd
      dn: uid="{{ item.username }}",dc=example,dc=com
      objectClass:
        - top
        - inetOrgPerson
        - posixAccount
        - shadowAccount
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