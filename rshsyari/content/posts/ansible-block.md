---
date: 2025-03-22T23:58:38+07:00
# description: ""
# image: ""
# lastmod: 2025-03-22
showTableOfContents: true
# tags: ["",]
title: "Ansible Block"
type: "post"
draft: false
---

Using Ansible in some case will require you to create logical groups of tasks. That can be done using `ansible block`, so let's talk about it.

## Basic usage
e.g. in the playbook below, we are configuring a AD Domain Services of a domain controller. The required information is taken from a JSON file, here is the example without `block`:
### Without block
```yaml
- name: Configure customers deployment
  hosts: windows
  gather_facts: false
  vars:
    data: "{{ lookup('file','/etc/ansible/customers.json') | from_json }}"
  tasks:
    - name: Create AD OU
      community.windows.win_domain_ou:
        name: "{{ item.name }}"
        path: "dc=customers,dc=com"
      loop: "{{ data }}"
      when: '"dc" in group_names'
      tags: dc

    - name: Create AD Group
      community.windows.win_domain_group:
        name: "{{ item.name }}"
        path: "ou={{ item.name }},dc=customers,dc=com"
      loop: "{{ data }}"
      when: '"dc" in group_names'
      tags: dc

    - name: Create AD User
      community.windows.win_domain_user:
        name: "{{ item.username }}"
        password: "{{ item.password }}"
        path: "ou={{ item.name }},dc=customers,dc=com"
        update_password: on_create
      loop: "{{ data }}"
      when: '"dc" in group_names'
      tags: dc

    - name: Create document root
      win_file:
        path: "C:/inetpub/wwwroot/{{ item.domain_prefix }}"
        state: directory
      loop: "{{ data }}"
      when: '"iis" in group_names'
      tags: iis

    - name: Create website content
      win_copy:
        dest: "C:/inetpub/wwwroot/{{ item.domain_prefix }}/index.html"
        content: "{{ item.message }}"
      loop: "{{ data }}"
      when: '"iis" in group_names'
      tags: iis

    - name: Configure virtualhost
      win_iis_website:
        hostname: "{{ item.domain_prefix }}.customers.com"
        name: "{{ item.name }}"
        physical_path: "C:/inetpub/wwwroot/{{ item.domain_prefix }}"
      loop: "{{ data }}"
      when: '"iis" in group_names'
      tags: iis
```
As you can probably see, the usage of `when` and `tag` in literally every tasks is redundant, when with `block` you only need it once. Unfortunately the case isn't the same for loops, `loop` cannot be used in `block` like you do with `when`, `delegate_to`, or `tags`. Now let me show you how it would look like with `block`:

### With block
```yaml
- name: Configure customers deployment
  hosts: windows
  gather_facts: false
  vars:
    data: "{{ lookup('file','/etc/ansible/customers.json') | from_json }}"
  tasks:
    - block:
      - name: Create AD OU
        community.windows.win_domain_ou:
          name: "{{ item.name }}"
          path: "dc=customers,dc=com"
        loop: "{{ data }}"

      - name: Create AD Group
        community.windows.win_domain_group:
          name: "{{ item.name }}"
          path: "ou={{ item.name }},dc=customers,dc=com"
        loop: "{{ data }}"

      - name: Create AD User
        community.windows.win_domain_user:
          name: "{{ item.username }}"
          password: "{{ item.password }}"
          path: "ou={{ item.name }},dc=customers,dc=com"
          update_password: on_create
        loop: "{{ data }}"
      when: '"dc" in group_names'
      tags: dc

    - block:
      - name: Create document root
        win_file:
          path: "C:/inetpub/wwwroot/{{ item.domain_prefix }}"
          state: directory
        loop: "{{ data }}"

      - name: Create website content
        win_copy:
          dest: "C:/inetpub/wwwroot/{{ item.domain_prefix }}/index.html"
          content: "{{ item.message }}"
        loop: "{{ data }}"

      - name: Configure virtualhost
        win_iis_website:
          hostname: "{{ item.domain_prefix }}.customers.com"
          name: "{{ item.name }}"
          physical_path: "C:/inetpub/wwwroot/{{ item.domain_prefix }}"
        loop: "{{ data }}"
      when: '"iis" in group_names'
      tags: iis
```
You can see it is much cleaner, and makes it more efficient because of lesser lines written. But besides code efficiency, it can be used for error handling too!
Introducing: `rescue` and `always`.

## Error handling
### Rescue usage
```yaml
 tasks:
   - name: Handle the error
     block:
       - name: Print a message
         debug:
           msg: 'This execute normally'

       - name: Force a failure
         command: /bin/false

       - name: Never print this
         debug:
           msg: 'This never execute'
     rescue:
       - name: Print when errors
         ansible.builtin.debug:
           msg: 'This executes because a failure exists!'
```
Ansible only runs `rescue` section after a task in `block` section returns a ‘failed’ state. Bad task definitions and unreachable hosts will not trigger the rescue block.

Let's say we want to use rescue in addition to our earlier playbook, we want that everytime there's an error (e.g. failed to promote and becoming domain controller), it would display the error through the terminal we run the playbook from.
```yaml
  tasks:
    - block:
          # Your code here
      rescue:
        - name: Display error
          debug: msg="{{ ansible_failed_result }}"
```
Lastly, there is `always` which will always run no matter the state of a previous task.

### Always usage
```yaml
  tasks:
    - block:
        - name: Accumulate success
          ansible.builtin.set_fact:
            result:
              host: "{{ inventory_hostname }}"
              status: "OK"
              interfaces: "{{ ansible_facts['interfaces'] }}"
      rescue:
        - name: Accumulate failure
          ansible.builtin.set_fact:
            result:
              host: "{{ inventory_hostname }}"
              status: "FAIL"
      always:
        - name: Will always run after the main block
          block:
            - name: Collect results
              set_fact:
                global_result: "{{ (global_result | default([])) + [hostvars[item].result }}"
              loop: "{{ ansible_play_hosts }}"

            - name: Classify results
              set_fact:
                result_ok: "{{ _global_result | selectattr('status', 'equalto', 'OK') | list }}"
                result_fail: "{{ _global_result | selectattr('status', 'equalto', 'FAIL') | list }}"
            
            - name: Display results OK
              debug: msg="{{ result_ok }}"
              when: (result_ok | length ) > 0

            - name: Display results FAIL
              debug: msg="{{ result_fail }}"
              when: (result_fail | length ) > 0
          delegate_to: localhost
          run_once: true
          tag: results
```
Regardless of the tasks state; failed or successful, the `always` section will ALWAYS run.

This `always` section collects the results saved in the variable `result` from the main block section. The classify results task sort it based on the status attributes it was given either by successully executed in the main `block` section or failed to execute in the `rescue` section.

Additionally, you can see that inside the `always` section, we are using nested `block`. Yes, you can use nested `block`, even inside the main `block` section if you need to. But of course having a `block` section inside of a `block` section is not the best practice there is.
