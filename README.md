# infra
Ansible project to bootstrap new servers

``` $ brew install ansible ```

``` $ ansible-playbook bootstrap.yml -e "host=<new host> hostname=<new hostname>" -K -i hosts ```
