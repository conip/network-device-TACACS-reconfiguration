### What is this repository for? ###

This repository has scripts for specific network device types in order to reconfigure TACACS server to new ISE MO environment. 

Device types covered:

* ASA
* NEXUS

### How do I get set up? ###

UK environment:

* script must be executed from "ens30ans01" 
* login there as admin and execute with su priviledges 
* path where scripts are located:

```
/home/admin/pkonitz/tacacs/
```
TACACS servers which are going to be configured are defined in host file vars:


```
isebackdoor_user: ensadmin
backdoor_pass: Alamakota$123
tacacs_shared_key: alamakota
ise_servers:
  - "192.168.101.99"
  - "170.118.82.40"
aaa_group_name: ISE_MOA
new_ise_local_username: testuser
new_ise_local_password: Alamakota$123
```


backdoor's credentials and TACACS shared key are stored in 

```
./vars/sensitive_vars.yml
```

## Usage examples:
### ASA
__1) script 1 - "asa-tacacs-change-v3.yml"__
```
ansible-playbook -i hosts-ens-asa-all asa-tacacs-change-v3.yml -f 10
```

script does the following:

* does PING test towards "ise_srv_1"
* checks HA and is executed only on ACTIVE one
* defines backdoor user (ensadmin)
* disables command authorization
* defines AAA server group "ISE-MOA"
* defines tacacs serves in the above group
* replaces "aaa authentication ssh console" with new AAA group

__2) script 2 - "asa-test-save-v3.yml"__
```
ansible-playbook -i hosts-ens-asa-all asa-test-save-v3.yml -f 10
```

script does the following:

* checks if AAA server group "ISE-MOA" already exists (if not it will not execute any further actions)
* removes the following commands

```
		aaa authentication http console .*
		aaa authentication enable console .* 
		aaa accounting command .*
		aaa authentication serial console .*
```

* adds new commands:

```
		aaa authentication http console ISE-MOA LOCAL
		aaa authentication enable console ISE-MOA LOCAL
		aaa accounting command ISE-MOA
		aaa authorization command ISE-MOA LOCAL
		aaa authentication serial console ISE-MOA LOCAL
```

* saves configuration 


### NEXUS

there are variables which need to be defined in the below file

```
./vars/sensitive_vars.yml

backdoor_user: ensadmin
backdoor_pass: Alamakota$123
tacacs_shared_key: alamakota
ise_servers:
  - "192.168.101.99"
  - "170.118.82.40"
aaa_group_name: ISE_MOA
new_ise_local_username: testuser
new_ise_local_password: Alamakota$123
```

You need to define the following:

* tacacs shared key - (becasue of that it is recommended to use ansible vault to encrypt that data)
* the list of servers (ise_servers) to be added by script to nexus configuration
* "aaa_group_name"
* local ISE credentials for testing purpose. Script uses these to execute 


```
	test aaa group ISE_MOA test-user test-pass
```


__Order of operation:

__1) script 1 - "nxos-1-tacacs-change.yml"__

* checks how many "aaa group server tacacs" are defined - if more than 1 - stops
* evaluates what is defined as "source-interface" and "use-vrf" as we want to use the same for new AAA definition
* adds new AAA group
* adds servers based on variable file
* test new AAA group 
* if user auth successfully adds the following config (aaa_group_name - taken from var file):

```
                aaa authentication login default fallback error local
                aaa authentication login default group {{ aaa_group_name }} local
                aaa authentication login console group {{ aaa_group_name }} local
                aaa accounting default group {{ aaa_group_name }}
                aaa authorization commands console group {{ aaa_group_name }} local
                aaa authorization commands default group {{ aaa_group_name }} local
                aaa authorization config-commands console group {{ aaa_group_name }} local
                aaa authorization config-commands default group {{ aaa_group_name }} local
```

__2) script 2 - "nxos-2-tacacs-cleanup.yml"__

* checks existing "aaa group server tacacs" definitions
* removes all except the one defined in var file (i.e ISE-MOA)
* removes all servers from all groups found earlier (if the same server is being used by new ISE-MOA and old one - it will be removed)