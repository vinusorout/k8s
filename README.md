k8s personal learning
## k8s install and use flexVolumen
To setup kubernetes cluster apply all steps from https://www.mirantis.com/blog/how-install-kubernetes-kubeadm/
NOTE: While applying calico plugin use the latest yaml from https://docs.projectcalico.org/getting-started/kubernetes/self-managed-onprem/onpremises

Check if flex volume installed!! kubernetes check for location /usr/libexec/kubernetes/kubelet-plugins/volume/exec, for flexvolume

```bash
ls /usr/libexec/kubernetes/kubelet-plugins/volume/exec
```

The plugin directory can be configured with the kubelet's --volume-plugin-dir parameter

