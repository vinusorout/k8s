k8s personal learning
## k8s install and use flexVolumen in minikube
To SSH in minikube:

```bash
minikube ssh
```
Check if flex volume installed!! kubernetes check for location /usr/libexec/kubernetes/kubelet-plugins/volume/exec, for flexvolume

```bash
ls /usr/libexec/kubernetes/kubelet-plugins/volume/exec
```

The plugin directory can be configured with the kubelet's --volume-plugin-dir parameter

