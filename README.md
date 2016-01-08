# infra
Ansible project to bootstrap new servers

```$ brew install python```
```$ pip install dopy six  # For digital_ocean.py inventory script```
```$ pip install ansible```

```$ ansible-playbook bootstrap.yml -e "host=<new host> hostname=<new hostname>"```
