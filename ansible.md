# Ansible
Here are some random notes regarding ansible.


## Ansible Facts
The following works with ansible 2.4.6.0 for sure. Didn't test w/ other versions.

Dump all ansible facts to JSON format.
```
ansible $(hostname) -i inventory/script.py -m setup | sed -e '1s/.*/{/' > variables.json
```

Dump just the ansible_local facts to JSON format.
```
ansible $(hostname) -i inventory/script.py -m setup -a "filter=ansible_local" | sed -e '1s/.*/{/' > variables.json
```

