``` Shell
sudo apt install git
sudo apt install python3-pip
sudo apt install python3-venv
sudo pip install ansible
```
Perhaps should have installed ansible-core

This installed:
- Python 3.9.2
- PIP
- Ansible 8.7.0

Para ejecutar:
``` Shell
ansible-pull -vv -i inventory -C psanio-basic-1 -U https://github.com/psanio/raspi-pull site.yml
```
