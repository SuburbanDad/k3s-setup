# k3s setup for pi4's

all nodes using rpi OS need a few baseline tweaks first:

 * enable cgroup support, edit /boot/cmdline.txt to include `cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory`
 * enable legacy iptables, issue `sudo update-alternatives --set iptables /usr/sbin/iptables-legacy`


to be sure all is well, run this diagnostic:
    `curl -sfL https://raw.githubusercontent.com/rancher/k3s/master/contrib/util/check-config.sh | sh -`

on kernels >5.3.0 it is safe to ignore these two errors:
    - CONFIG_NF_NAT_IPV4: missing (fail)
    ...
	- CONFIG_NF_NAT_NEEDED: missing (fail)


## Setup master node:
`curl -sfL https://get.k3s.io | sh -`


* save yourself a shitton of annoyance and add pi to root group.  now no more sudo to interact with k3s:
	`sudo usermod -a -G root p`
	`sudo chmod 660 /etc/rancher/k3s/k3s.yaml`
* once settled, get node TOKEN:
	`sudo cat /var/lib/rancher/k3s/server/node-token`




## Setup worker nodes:
    export NODE_TOKEN="whatever node token"
	`curl -sfL https://get.k3s.io | K3S_URL=https://pi1.home:6443 K3S_TOKEN=$NODE_TOKEN sh -`




## Installing k8s dashboard:
https://rancher.com/docs/k3s/latest/en/installation/kube-dashboard/

* zsh is not happy with the VERSION_KUBE_DASHBOARD envar set here, use bash:
	```
		GITHUB_URL=https://github.com/kubernetes/dashboard/releases
		VERSION_KUBE_DASHBOARD=$(curl -w '%{url_effective}' -I -L -s -S ${GITHUB_URL}/latest -o /dev/null | sed -e 's|.*/||')
		sudo k3s kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/${VERSION_KUBE_DASHBOARD}/aio/deploy/recommended.yaml
	```
* from dash/ :
	`sudo k3s kubectl create -f dashboard.admin-user.yaml -f dashboard.admin-user-role.yaml`

* get token:
  `sudo k3s kubectl -n kubernetes-dashboard describe secret admin-user-token | grep ^token`


## Setup helm3:
`curl -sfL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash -`
`helm repo add stable https://kubernetes-charts.storage.googleapis.com/`

`helm repo update`

* run this and/or add this to your shell profile:
	`export KUBECONFIG=/etc/rancher/k3s/k3s.yaml`


## Install prometheus:
* use an arm64 friendly kube-state-metrics:

	`helm install prometheus stable/prometheus --set kube-state-metrics.image.repository=carlosedp/kube-state-metrics`

* maybe should create this image in suburbandad repo to ensure its existence and evolution

## Install grafana:
* if you want autodiscovery of sources, it is a pain in the ass since `kiwigrid/k8s-sidecar` only has amd64 builds, if you want to use this:
	`helm install grafana stable/grafana --set sidecar.datasources.enabled=true --set persistence.enabled=true --set persistence.type=pvc --set persistence.size=500Mi`
  you will have to manually edit the deployment and change the sidecar image to `suburbandad/k8s-sidecar:latest`   

  otherwise, you will have to manually add prometheus data source to grafana after startup but can use this:
    `helm install grafana stable/grafana --set persistence.enabled=true --set persistence.type=pvc --set persistence.size=500Mi`



## Getting kube context setup on remote machine:

copy the relevant bits from /etc/rancher/k3s/ into your local ~/.kube/config


## Setup ingress:
* k3s uses traefik, lean and easy
* for grafana ingress see ingress/grafana.yaml


## k3s prysmatic helm chart
The goal is to run an eth2 ecosystem on this cluster; using the helm chart in eth2/prysmatic/helm 
* StatefulSet
* using volumeClaimTemplates 
* using node anti-affinity
* from beacon dir, run 
	`helm install prysm-beacon .`





## TODO:
left off with grafana and prometheus, trying to get the helm chart pods to automatically get their metrics endpoints scraped
  * maybe needs a service ?
  * dropped grafana, when redeploying with persistence=true, it no longer wants to start and is unhappy. :shrug:
* setup a service for statefulset pods
* create and publish a validator image
  * work on keys/secrets/etc for validator













## research/testing/troubleshooting:

### note on HA configuration:

https://rancher.com/docs/k3s/latest/en/installation/ha/

requires an external datastore, mysql or sqlite.  or if the embedded HA sqlite db stuff is GA, maybe use that






### persistent volume testing
https://rancher.com/docs/k3s/latest/en/storage/

* rancher out of the box offers local storage
* to use `local-path` persistence, we only have to define the claim.
  * not seeing a way to direct it to a specific node at creation
  * it seems to resolve as soon as the pod requests it
  * once there, it should always be there, and presumably impacts pod scheduling
  * net/net, deployment and/or pod definitions should implicitly place the PVCs whereever the pods
    get scheduled at first (which node).  For the example/POC, we make sure the CPU request is
    high enough that it forces the pods to land of different hosts. 
* see POC defs of 3 data PVs and beacon test pods 
  /persistence/
    pvc.yaml
    beacon-test-pods.yaml
* persistent volume resources in k8s have the node's hostpath in (Source -> hostPath)
 e.g. `/var/lib/rancher/k3s/storage/pvc-dbd53af7-4729-45ea-9d50-f5e1230d23ee`











troubleshooting:
----------------
    

    problem starting k3s server:
    -----------------------------
    syslog showing a lot of:

		Jun 12 00:02:12 pi1 k3s[6083]: time="2020-06-12T00:02:12.941783264-07:00" level=error msg="Failed to find memory cgroup, you may need to add \"cgroup_memory=1 cgroup_enable=memory\" to your linux cmdline (/boot/cmdline.txt on a Raspberry Pi)"
		Jun 12 00:02:12 pi1 k3s[6083]: time="2020-06-12T00:02:12.941881213-07:00" level=fatal msg="failed to find memory cgroup, you may need to add \"cgroup_memory=1 cgroup_enable=memory\" to your linux cmdline (/boot/cmdline.txt on a Raspberry Pi)"

	but it looks like the error message is all that is necessary.  update /boot/cmdline.txt to include the cgroup_memory flags
    sources indicate that raspbian should add three flags to cmdline.txt:  cgroup_enable=cpuset and cgroup_memory=1 cgroup_enable=memory


    problems starting k3s agent/worker:
    -----------------------------------
	Jun 12 08:20:53 pi2 k3s[479]: time="2020-06-12T08:20:53.320014860-07:00" level=error msg="failed to get CA certs at https://127.0.0.1:36233/cacerts: Get https://127.0.0.1:36233/cacerts: read tcp 127.0.0.1:45818->127.0.0.1:36233: read: connection reset by peer"

	for me this was a dns screwup.  agent was installed with an improper server hostname

    debug host/node environment:
    ----------------------------
    `kubectl run debug2 -it --image ubuntu:18.04 -- bash`
