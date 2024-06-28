# ansible cheatsheet
## Playbooks
```sh
ansible-playbook <YAML>                   # Run on all hosts defined
ansible-playbook <YAML> -f 10             # Run 10 hosts parallel
ansible-playbook <YAML> --verbose         # Verbose on successful tasks
ansible-playbook <YAML> -C                # Test run
ansible-playbook <YAML> -C -D             # Dry run
ansible-playbook <YAML> -l <host>         # Run on single host
```
## Run Infos
```sh
ansible-playbook <YAML> --list-hosts
ansible-playbook <YAML> --list-tasks
```

## Syntax Check
```sh
ansible-playbook --syntax-check <YAML>
```

## Inventory 
```sh
ansible-inventory --graph
```

## Remote Execution
```sh
ansible all -m ping
```

## Execute arbitrary commands
```sh
ansible <hostgroup> -a <command>
ansible all -a "ifconfig -a"
ansible all -a "hostname"
```

## Debugging
List facts and state of a host
```
ansible <host> -m setup                            # All facts for one host
ansible <host> -m setup -a 'filter=ansible_eth*'   # Only ansible fact for one host
ansible all -m setup -a 'filter=facter_*'          # Only facter facts but for all hosts
```

## Save facts to per-host files in /tmp/facts
```sh
ansible all -m setup --tree /tmp/facts
```

## Tutorials Ansible fr
[Nice set of tutos in french](https://gitlab.com/xavki/presentation-ansible-fr)
