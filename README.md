# Overview #
This tutorial shows how to secure access to the arcade game Pac-Man using Oauth2-proxy, Dex and an OpenLDAP server.

# Prerequisites #
## Docker and Helm ##

* Docker - https://docs.docker.com/get-docker/​

* Helm - https://helm.sh/docs/intro/install/

## Kind ##
### Installation

Kind - https://kind.sigs.k8s.io/​

* Macbook:
```
brew install kind
```

If you have “go 1.17+” installed​
```
go install sigs.k8s.io/kind@v0.12.0​
```

For older versions of go​
```
GO111MODULE="on" go get sigs.k8s.io/kind@v0.12.0
```

Locate the kind command​
```
ls $(go env GOPATH)/bin/kind​
```

https://go.dev/dl/​

https://kind.sigs.k8s.io/#installation-and-usage

### Create up a Kind cluster
```
kind create cluster --name <Name of the cluster>
```
## OpenLDAP - ldap Utils​
* Macbook:​
```
brew install openldap​
```
* Debian/Ubuntu:​
```
apt-get install ldap-utils
```
----
# Tutorials #
## OpenLDAP service ##
Create Namespace and Secret​
```
kubectl create ns openldap​
kubectl create secret generic openldap --from-literal=adminpassword=adminpassword --from-literal=users=productionadmin,productionbasic,productionconfig --from-literal=passwords=testpasswordadmin,testpasswordbasic,testpasswordconfig -n openldap
```

Create Deployment​
```
cd openldap​
kubectl create -n openldap -f openldap-deployment.yaml​
```

Create Service
```
kubectl create -n openldap -f openldap-service.yaml​
```

Port-forward
```
kubectl port-forward service/openldap -n openldap 1389:1389
```

Add a Group​
```
ldapadd -x -H ldap://127.0.0.1:1389 -D "cn=admin,dc=example,dc=org" -w adminpassword -f pacman-admin-group.ldif
```

OpenLDAP - Search
```
ldapsearch -x -H ldap://127.0.0.1:1389 -b dc=example,dc=org -D "cn=admin,dc=example,dc=org" -w adminpassword​
```

## Dex ##
Install via Helm
```
kubectl create ns dex​
helm repo add dex https://charts.dexidp.io​
helm repo update​
```

Update `bindPW` in dex-values.yaml
```
helm install dex dex/dex -n dex -f dex-values.yaml
```

Verify installation
```
helm status dex -n dex
```

Update network
```
kubectl port-forward service/dex -n dex 5556:5556
sudo vi /etc/hosts
```
Add
```
127.0.0.1 dex.dex
```

## OAuth2 Proxy ##
Create Deployment and Service
```
cd oauth2-proxy
kubectl create ns pacman
kubectl create -f oauth2-proxy-deployment.yaml -n pacman
kubectl create -f oauth2-proxy-service.yaml -n pacman
```
Update network
```
sudo vi /etc/hosts
```
Add
```
127.0.0.1 oauth2-proxy.pacman
```

## Pac-man ##
Install via Helm
```
helm repo add pacman https://shuguet.github.io/pacman/
helm repo update
helm install pacman pacman/pacman -n pacman
```

Verify installation
```
helm status pacman -n pacman
watch kubectl get pod -n pacman
```

Update pacman service
```
kubectl patch svc pacman -n pacman --type='json' -p='[{"op": "replace", "path": "/spec/ports/0/targetPort", "value":4180}]'
kubectl patch svc pacman -n pacman --patch '{"spec": {"selector": {"k8s-app": "oauth2-proxy"}}}'
```

Deploy pacman actual service
```
cd pacman
kubectl create –f pacman-actual-service.yaml -n pacman
```

Port-forward
```
kubectl port-forward service/pacman -n pacman 8080:80
```







​

​

​


​

​