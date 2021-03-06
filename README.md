# rpi-k8s-ansible
Raspberry PI's running Kubernetes deployed with Ansible

Masters:
- rPi 3b+ x1

Workers:
- rPi 3b+ x2
- rPi 3b x4

CNI: Flannel (Weave support is there, but it crashes and reboot's the rPi's, see below)

## Preparing an SD card on Linux
```
# Write the image to the SD card, please use at least 2018-04-18 if you want to use WiFi
$ sudo dd if=YYYY-MM-DD-raspbian-stretch-lite.img of=/dev/sdX bs=16M status=progress

# Provision wifi settings on first boot of the board with usb with password pi/raspberry
sudo raspi-config
# setup wifi
# enable ssh

```

## Updating cluster.yml to match your environment
This is where there individual rPi's are set to be a master or a slave.  
I have not changed any passwords or configured SSH keys as this cannot be easily done with a headless rPi setup.  
I am currently using DHCP static assignment to ensure each PI's MAC address is given the same IP address.  
Update the file as required for your specific setup.

# Example Playbooks
## apt-get upgrade
```
ansible-playbook -i cluster.yml playbooks/upgrade.yml
```

## rPi Overclocks
Ensure you update cluster.yml with the correct children mappings for each rpi model
```
ansible-playbook -i cluster.yml playbooks/overclock-rpis.yml
```

## Install k8s
With the below commands, you need to include the master node (node00) in all executions for the token to be set correctly.
```
# Bootstrap the master and all slaves
ansible-playbook -i cluster.yml site.yml

# Bootstrap a single slave (node05)
ansible-playbook -i cluster.yml site.yml -l node00,node05

# When running again, feel free to ignore the common tag as this will reboot the rpi's
ansible-playbook -i cluster.yml site.yml --skip-tags common
```

# Running stuff on the cluster!
## Example HA web service
This example will create a, Ingress Controller, Service, Deployment and 5x Nginx pods that will print out the node name they are running on!
```
pi@node00:~ $ kubectl get po
No resources found.
pi@node00:~ $ kubectl apply -f https://raw.githubusercontent.com/kenneyhe/rpi-k8s-ansible/master/kubernetes/example_ha_website/web/ingress-controller-base.yaml
namespace "ingress-nginx" configured
deployment.extensions "default-http-backend" configured
service "default-http-backend" unchanged
configmap "nginx-configuration" unchanged
configmap "tcp-services" unchanged
configmap "udp-services" unchanged
serviceaccount "nginx-ingress-serviceaccount" unchanged
clusterrole.rbac.authorization.k8s.io "nginx-ingress-clusterrole" configured
role.rbac.authorization.k8s.io "nginx-ingress-role" unchanged
rolebinding.rbac.authorization.k8s.io "nginx-ingress-role-nisa-binding" unchanged
clusterrolebinding.rbac.authorization.k8s.io "nginx-ingress-clusterrole-nisa-binding" configured
deployment.extensions "nginx-ingress-controller" configured
pi@node00:~ $ kubectl apply -f  https://raw.githubusercontent.com/kenneyhe/rpi-k8s-ansible/master/kubernetes/example_ha_website/web/ingress-service-nodeport.yaml

pi@node00:~ $ kubectl apply -f https://raw.githubusercontent.com/kenneyhe/rpi-k8s-ansible/master/kubernetes/example_ha_website/web/ingress-service-deployment-web.yaml
service "webserver-service" created
deployment.apps "webserver-deployment" created
pi@node00:~ $ kubectl get pods
NAME                                    READY     STATUS              RESTARTS   AGE
webserver-deployment-7c7948b97f-q9bpk   0/1       ContainerCreating   0          5s
webserver-deployment-7c7948b97f-s95tp   0/1       ContainerCreating   0          5s
webserver-deployment-7c7948b97f-tls8n   0/1       ContainerCreating   0          5s
pi@node00:~ $ kubectl get po
NAME                                    READY     STATUS    RESTARTS   AGE
webserver-deployment-7c7948b97f-q9bpk   1/1       Running   0          28s
webserver-deployment-7c7948b97f-s95tp   1/1       Running   0          28s
webserver-deployment-7c7948b97f-tls8n   1/1       Running   0          28s
```

We can then simulate node failure by shutting down a node.  
It took around 5 minutes for k8s to detect the node failure and respond by rescheudling the pod on another node.  
Currently investigating how we can speed this up.
```
pi@node03:~ $ sudo shutdown -h now
Connection to 192.168.58.63 closed by remote host.
Connection to 192.168.58.63 closed.

pi@node00:~ $ kubectl get po -o wide
NAME                                    READY     STATUS    RESTARTS   AGE       IP           NODE
webserver-deployment-7c7948b97f-q9bpk   1/1       Running   0          23m       10.244.1.4   node04
webserver-deployment-7c7948b97f-s95tp   1/1       Running   0          23m       10.244.4.6   node02
webserver-deployment-7c7948b97f-tls8n   1/1       Running   0          23m       10.244.3.6   node03

pi@node00:~ $ kubectl get nodes
NAME      STATUS     ROLES     AGE       VERSION
node00    Ready      master    2d        v1.10.1
node01    Ready      <none>    2d        v1.10.1
node02    Ready      <none>    2d        v1.10.1
node03    NotReady   <none>    2d        v1.10.1
node04    Ready      <none>    2d        v1.10.1

pi@node00:~ $ kubectl get node node03 --output=yaml
...
  - lastHeartbeatTime: 2018-04-27T12:28:19Z
    lastTransitionTime: 2018-04-27T12:29:01Z
    message: Kubelet stopped posting node status.
    reason: NodeStatusUnknown
    status: Unknown
    type: Ready
...

pi@node00:~ $ kubectl get deployment webserver-deployment --output=yaml
...
  - lastTransitionTime: 2018-04-27T12:29:01Z
    lastUpdateTime: 2018-04-27T12:29:01Z
    message: Deployment does not have minimum availability.
    reason: MinimumReplicasUnavailable
    status: "False"
    type: Available
  observedGeneration: 1
  readyReplicas: 2
  replicas: 3
  unavailableReplicas: 1
  updatedReplicas: 3
...

pi@node00:~ $ kubectl get po -o wide
NAME                                    READY     STATUS    RESTARTS   AGE       IP           NODE
webserver-deployment-7c7948b97f-lld6r   1/1       Running   0          47s       10.244.2.4   node01
webserver-deployment-7c7948b97f-q9bpk   1/1       Running   0          27m       10.244.1.4   node04
webserver-deployment-7c7948b97f-s95tp   1/1       Running   0          27m       10.244.4.6   node02
webserver-deployment-7c7948b97f-tls8n   1/1       Unknown   0          27m       10.244.3.6
```

# Extra misc commands
```
# Shutdown all nodes
ansible -i cluster.yml -a "shutdown -h now" all
```

Reference:

# single node master node single cluster
[![asciicast](https://asciinema.org/a/PPT0bc7em7NPr6vcJfzks6Ywe.svg)](https://asciinema.org/a/PPT0bc7em7NPr6vcJfzks6Ywe)

# hypriot example
https://blog.hypriot.com/post/setup-kubernetes-raspberry-pi-cluster/

# kube dashboard
https://github.com/kubernetes/dashboard

# over internet service
https://www.raspberrypi.org/documentation/remote-access/access-over-Internet/
