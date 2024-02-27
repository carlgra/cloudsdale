# cloudsdale

fresh install of ubuntu on 4 nodes - twilight, celestia, luna, crankydoodle

## Set up VPN

install tailscale on each host

```
curl -fsSL https://tailscale.com/install.sh | sh

sudo tailscale up
```

(go to site it suggests and auth)

enable ssh 
```
tailscale up --ssh
```

node should now be accessible via ssh from other tailscale hosts on VPN

## install k3s on nodes

on local machine
install pip package manager

```
python3 -m pip install --user ansible

pip3 install --upgrade pip
```

might have to add pip to PATH
```
export PATH=$HOME/bin:/usr/local/bin:$HOME/go/bin:$PATH/Library/Python/3.11/bin:$PATH
```

get ansible k3s
```
git clone https://github.com/k3s-io/k3s-ansible.git
```

 
follow instructions on k3s ansible.  edit inventory.yml to configure hostnames, ansible user (ubuntu in my case)

```
ansible-playbook playbook/site.yml -i inventory.yml
```

the ~/.kube/config file from the control node host is copied to your local ~/.kube/config may need to edit this if you did not set up etc/hosts on the control node.

e.g. 
```
    server: https://twilight:6443
```

