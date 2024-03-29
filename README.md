
 cloudsdale

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

if it doesnt bring it back can scp i
````
scp -r ubuntu@twilight:/home/ubuntu/.kube .
```
e.g. 
```
    server: https://twilight:6443
```
check k3s is running on all nodes from master node (twilight)

```
kubectl config use-context k3s-ansible
kubectl get nodes
```


## K9s running locally

install kubernetes cli locally and copy config to localhost from control node

```
brew install kubernetes-cli
scp -r ubuntu@twilight:/home/ubuntu/.kube .
cp -r .kube $HOME/
```
edit config file for master node name in server

run k9s to check connectivity


# Installing ArgoCD 
e.g.  https://medium.com/be-tech-with-santander/from-git-to-kubernetes-in-10-minutes-with-argocd-3027a2d5ea62
to uninstall...
```
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

create argocd namespace from local
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```


ddexpose to local host insecurely for now 
TODO: set up ingress - https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/

```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

port forward
```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

get and change the first time password
```
argocd admin initial-password -n argocd
```

# create git repo for K8s app configuration (this repo)

```
h repo create --public cloudsdale -y && cd cloudsdale
echo "#Cloudsdale." > Readme.md
git add Readme.md
git commit -m "Added Readme.md" && git push --set-upstream origin master

```

create a live branch

```
git branch live
git push --set-upstream origin live
```


```
git checkout master
curl -kSs https://raw.githubusercontent.com/kubernetes/examples/master/guestbook/all-in-one/guestbook-all-in-one.yaml -o guestbook_app.yaml
git add guestbook_app.yaml
git commit -m "Added guestbook_app.yaml"
git push --set-upstream origin main
```

set address of git repo

```
 HTTPS_REPO_URL=$(git remote show origin |  sed -nr 's/.+Fetch URL: git@(.+):(.+).git/https:\/\/\1\/\2.git/p')
```

create new namespace

```
kubectl create namespace cloudsdale
```

deploy app

```
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cloudsdale
  namespace: argocd
spec:
  destination:
    namespace: cloudsdale
    server: https://kubernetes.default.svc
  project: default
  source:
    repoURL: $HTTPS_REPO_URL
    path: .
    targetRevision: main
EOF
```

looking at Ingress to open up argo cd ui

It turns out that Traefik has a dashboard. Here’s how to access it via kubectl port-forward.

check ingress
```
kubectl get ingress -A
```
kubectl --namespace kube-system port-forward deployments/traefik 9000:9000 &
Then navigate to http://localhost:9000/dashboard/.

If you want to make this more permanent by using an Ingress, see https://k3s.rocks/traefik-dashboard/.

READ THIS NEXT
https://doc.traefik.io/traefik/providers/kubernetes-crd/


installing metallb from turingpi docs

 helm repo add metallb https://metallb.github.io/metallb
helm upgrade --install metallb metallb/metallb --create-namespace \
--namespace metallb-system --wait



cat << 'EOF' | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.0.0.70-10.0.0.80
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
  - default-pool
EOF

- DIDNT FINISH THIS


tried this
https://doc.traefik.io/traefik/getting-started/quick-start-with-kubernetes/

files in code/traefik
 this seems to work for the whoami service at least

iused the tailscale IP address from login report on twilight
http://100.91.183.30:8080/dashboard/#/

works to get to traefik dashboard :)


# HELM DASHBOARD
https://github.com/komodorio/helm-dashboard

on localhost

```
helm plugin install https://github.com/komodorio/helm-dashboard.git
helm plugin update dashboard
helm dashboard
```
-- to uninstall
helm plugin uninstall dashboard


l
