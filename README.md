# Overview #
This tutorial shows how to secure access to the arcade game Pac-Man using Oauth2-proxy, Dex and an OpenLDAP server - without requiring code changes to the Pac-Man app itself.

# Prerequisites #

In order to complete this tutorial, you will need an environment with the following prerequisites.

---

> **NOTE:**  The current version of this tutorial only works on x86 based platforms.

---

* [(macOS Only) Homebrew](#homebrew) - Package manager used to install prereqs
* [(Windows Only) Chocolatey](#chocolatey) - Package manager used to install prereqs
* [git](#git) - Used to clone the Pac-Man application
* [docker](#docker) - Container runtime
* [kind](#kind) - Running a local Kubernetes cluster using Docker container “nodes”
* [kubectl](#kubectl) - Kubernetes command-line tool
* [helm](#helm) - Kubernetes package manager
* [openldap](#openldap) - Used to populate OpenLDAP instance with user/group data


## Homebrew

Run the following in your Terminal to install ```brew``` (from [brew.sh](https://brew.sh)):

```/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"```

## Chocolatey

Follow the linked instructions from [chocolatey.org](https://chocolatey.org/install#individual) to install ```choco```.
## git

* macOS: ```brew install git```
* Windows: ```choco install git```
* Linux: [Distro-specific instructions](https://git-scm.com/download/linux)

Alternatively, you can install [GitHub Desktop](https://desktop.github.com/).

## docker

* macOS: ```brew install docker```
* Windows: ```choco install docker-desktop```
* Linux: [Distro-specifc instructions](https://docs.docker.com/engine/install/)

Alternatively, you can install [Docker Desktop](https://www.docker.com/get-started/).

## kind

* macOS: ```brew install kind```
* Windows: ```choco install kind```
* Linux (from [kind.sigs.k8s.io](https://kind.sigs.k8s.io/#installation-and-usage)):
  ```
  curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.13.0/kind-linux-amd64
  chmod +x ./kind
  mv ./kind /some-dir-in-your-$PATH/kind
  ```

## kubectl

* macOS: ```brew install kubectl```
* Windows: ```choco install kubernetes-cli```
* Linux: [Distro-specific instructions](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

## helm

* macOS: ```brew install helm```
* Windows: ```choco install kubernetes-helm```
* Linux: [Distro-specific instructions](https://helm.sh/docs/intro/install/)

## openldap

* macOS: ```brew install openldap```
* Windows: ```choco install openldap```
* Debian/Ubuntu: ```apt-get install ldap-utils```
* RedHat/CentOS: ```yum -y install openldap```

# Tutorial
The following steps correspond to the live tutorial walkthrough, which will provide great insight into the individual steps.
## Environment Setup
1. Create your Kind K8s cluster
    ```
    kind create cluster --name <Name of the cluster>
    ```
1. Verify `kubectl` context matches your new Kind cluster (i.e., **kind-\<Name of the cluster>**)
   ```
   kubectl config current-context
   ```
1. Clone repository

   ```
   git clone https://github.com/onkarbhat/secure-pacman.git
   ```
1. Change working directory to *secure-pacman*
## OpenLDAP

1. Create Namespace and Secret​
    ```
    kubectl create ns openldap
    kubectl create secret generic openldap --from-literal=adminpassword=adminpassword --from-literal=users=productionadmin,productionbasic,productionconfig --from-literal=passwords=testpasswordadmin,testpasswordbasic,testpasswordconfig -n openldap
    ```

1. Create Deployment​
    ```
    cd openldap
    kubectl create -n openldap -f openldap-deployment.yaml
    ```

1. Create Service
    ```
    kubectl create -n openldap -f openldap-service.yaml
    ```

1. Verify installation
    ```
    watch kubectl get pod -n openldap
    ```

    Wait for listed pod to be *Ready/Running*, press `Ctrl+C` and proceed to the next step.

2. (In a separate terminal) Initiate *service/openldap* port-forward
    ```
    kubectl port-forward service/openldap -n openldap 1389:1389
    ```

3. Add a Group​
    ```
    ldapadd -x -H ldap://127.0.0.1:1389 -D "cn=admin,dc=example,dc=org" -w adminpassword -f pacman-admin-group.ldif
    ```

4. Verify LDIF Import
    ```
    ldapsearch -x -H ldap://127.0.0.1:1389 -b dc=example,dc=org -D 'cn=admin,dc=example,dc=org' -w adminpassword
    ```

## Dex

1. Add Dex repo to Helm
    ```
    helm repo add dex https://charts.dexidp.io
    helm repo update dex
    cd ../dex
    ```

1. Update `bindPW` in *dex-values.yaml* to match imported admin user

1. Install Dex via Helm
    ```
    kubectl create ns dex
    helm install dex dex/dex -n dex -f dex-values.yaml
    ```

1. Verify installation
    ```
    helm status dex -n dex
    watch kubectl get pod -n dex
    ```

    Wait for listed pod to be *Ready/Running*, press `Ctrl+C` and proceed to the next step.

1. (In a separate terminal) Initiate *service/dex* port-forward
    ```
    kubectl port-forward service/dex -n dex 5556:5556
    ```

## OAuth2 Proxy
1. Create Deployment and Service
    ```
    cd ../oauth2-proxy
    kubectl create ns pacman
    kubectl create -f oauth2-proxy-deployment.yaml -n pacman
    kubectl create -f oauth2-proxy-service.yaml -n pacman
    ```

1. Verify installation
    ```
    watch kubectl get pod -n pacman
    ```

    Wait for listed pods to be *Ready/Running*, press `Ctrl+C` and proceed to the next step.

1. (In a separate terminal) Initiate *service/oauth2-proxy* port-forward
    ```
    kubectl port-forward service/oauth2-proxy -n pacman 4180:4180
    ```

1. Add *dex.dex* and *oauth2-proxy.pacman* entry into *hosts* file:

   * Linux/macOS: `sudo vi /etc/hosts`
   * Windows: `notepad C:\windows\system32\drivers\etc\hosts`

    Add the following entry, save, and close:
    ```
    127.0.0.1 dex.dex
    127.0.0.1 oauth2-proxy.pacman
    ```
## Pac-man
1. Install via Helm
    ```
    helm repo add pacman https://shuguet.github.io/pacman/
    helm repo update pacman
    helm install pacman pacman/pacman -n pacman
    ```

1. Verify installation
    ```
    watch kubectl get pod -n pacman
    ```

    Wait for listed pods to be *Ready/Running*, press `Ctrl+C` and proceed to the next step.

1. (In a separate terminal) Initiate *service/pacman* port-forward
    ```
    kubectl port-forward service/pacman -n pacman 9090:80
    ```
1. Open your browser to: http://127.0.0.1:9090/ and attempt to login to your application

1. Patch the *service/pacman* configuration to use OAuth-Proxy port and selector
    ```
    kubectl patch svc pacman -n pacman --type='json' -p='[{"op": "replace", "path": "/spec/ports/0/targetPort", "value":4180}]'
    kubectl patch svc pacman -n pacman --type='json' -p='[{"op": "replace", "path": "/spec/selector", "value":{"k8s-app": "oauth2-proxy"}}]'
    ```

1. Stop (`Ctrl+C`) and restart *service/pacman* port-forward
    ```
    kubectl port-forward service/pacman -n pacman 9090:80
    ```
1. Open your browser again to: http://127.0.0.1:9090/

1. Create Service
   ```
   kubectl create -f pacman-actual-service.yaml -n pacman
   ```

1. Stop (`Ctrl+C`) and restart *service/pacman* port-forward a final time
   ```
   kubectl port-forward service/pacman -n pacman 9090:80
   ```

1. Open your browser again to: http://127.0.0.1:9090/

1. Play Pac-man!



​

​

​


​

​