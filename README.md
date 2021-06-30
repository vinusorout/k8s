k8s personal learning

## Both flexVolume and csi are out of tree plugins, for some clouds like aws, gce, vmware etc kubernetes provides some inbuilt or in-tree plugins.
## k8s install and use flexVolume
To setup kubernetes cluster apply all steps from https://www.mirantis.com/blog/how-install-kubernetes-kubeadm/
NOTE: While applying calico plugin use the latest yaml from https://docs.projectcalico.org/getting-started/kubernetes/self-managed-onprem/onpremises

Check if flex volume installed!! kubernetes check for location /usr/libexec/kubernetes/kubelet-plugins/volume/exec, for flexvolume

```bash
ls /usr/libexec/kubernetes/kubelet-plugins/volume/exec
```
NOTE: If you use caico networking than, calico also install a flex volumen "uds" at /usr/libexec/kubernetes/kubelet-plugins/volume/exec/nodeagent~uds
The plugin directory can be configured with the kubelet's --volume-plugin-dir parameter

Now to create a flexvolume cifs plugin, your shell script or binary should implement following methods:

```bash
flexvolume_driver mount # mounts volume to a directory in the pod
# expected output:
{
  "status": "Success"/"Failure"/"Not supported",
  "message": "<Reason for success/failure>",
}
flexvolume_driver unmount # unmounts volume from a directory in the pod
# expected output:
{
  "status": "Success"/"Failure"/"Not supported",
  "message": "<Reason for success/failure>",
}
flexvolume_driver init # initializes the plugin
# expected output:
{
  "status": "Success"/"Failure"/"Not supported",
  "message": "<Reason for success/failure>",
  // defines if attach/detach methods are supported
  "capabilities":{"attach": True/False}
}
```

There is a simple fstab plugin as described https://github.com/fstab/cifs, we will use this plugin as our flex volulme
fstab shell script is:

```shell
#!/bin/bash

set -u

# ====================================================================
# Example configuration:
# ====================================================================
# --------------------------------------------------------------------
# secret.yml:
# --------------------------------------------------------------------
# apiVersion: v1
# kind: Secret
# metadata:
#   name: cifs-secret
#   namespace: default
# type: fstab/cifs
# data:
#   username: 'ZXhhbXBsZQo='
#   password: 'c2VjcmV0Cg=='
#
# --------------------------------------------------------------------
# pod.yml:
# --------------------------------------------------------------------
# apiVersion: v1
# kind: Pod
# metadata:
#   name: busybox
#   namespace: default
# spec:
#   containers:
#   - name: busybox
#     image: busybox
#     command:
#       - sleep
#       - "3600"
#     imagePullPolicy: IfNotPresent
#     volumeMounts:
#     - name: test
#       mountPath: /data
#   volumes:
#   - name: test
#     flexVolume:
#       driver: "fstab/cifs"
#       fsType: "cifs"
#       secretRef:
#         name: "cifs-secret"
#       options:
#         networkPath: "//example-server/backup"
#         mountOptions: "dir_mode=0755,file_mode=0644,noperm"
# --------------------------------------------------------------------

# Uncomment the following lines to see how this plugin is called:
# echo >> /tmp/cifs.log
# date >> /tmp/cifs.log
# echo "$@" >> /tmp/cifs.log

init() {
	assertBinaryInstalled mount.cifs cifs-utils
	assertBinaryInstalled jq jq
	assertBinaryInstalled mountpoint util-linux
	assertBinaryInstalled base64 coreutils
	echo '{ "status": "Success", "message": "The fstab/cifs flexvolume plugin was initialized successfully", "capabilities": { "attach": false } }'
	exit 0
}

assertBinaryInstalled() {
	binary="$1"
	package="$2"
	if ! which "$binary" > /dev/null ; then
		errorExit "Failed to initialize the fstab/cifs flexvolume plugin. $binary command not found. Please install the $package package."
	fi
}

errorExit() {
	if [[ $# -ne 1 ]] ; then
		echo '{ "status": "Failure", "message": "Unknown error in the fstab/cifs flexvolume plugin." }'
	else
		jq -Mcn --arg message "$1" '{ "status": "Failure", "message": $message }'
	fi
	exit 1
}

doMount() {
	if [[ -z ${1:-} || -z ${2:-} ]] ; then
		errorExit "cifs mount: syntax error. usage: cifs mount <mount dir> <json options>"
	fi
	mountPoint="$1"
	shift
	json=$(printf '%s ' "${@}")
	if ! jq -e . > /dev/null 2>&1 <<< "$json" ; then
		errorExit "cifs mount: syntax error. invalid json: '$json'"
	fi
	networkPath="$(jq --raw-output -e '.networkPath' <<< "$json" 2>/dev/null)"
	if [[ $? -ne 0 ]] ; then
		errorExit "cifs mount: option networkPath missing in flexvolume configuration."
	fi
	mountOptions="$(jq --raw-output -e '.mountOptions' <<< "$json" 2>/dev/null)"
	if [[ $? -ne 0 ]] ; then
		errorExit "cifs mount: option mountOptions missing in flexvolume configuration."
	fi
	cifsUsernameBase64="$(jq --raw-output -e '.["kubernetes.io/secret/username"]' <<< "$json" 2>/dev/null)"
	if [[ $? -ne 0 ]] ; then
		errorExit "cifs mount: username not found. the flexVolume definition must contain a secretRef to a secret with username and password."
	fi
	cifsPasswordBase64="$(jq --raw-output -e '.["kubernetes.io/secret/password"]' <<< "$json" 2>/dev/null)"
	if [[ $? -ne 0 ]] ; then
		errorExit "cifs mount: password not found. the flexVolume definition must contain a secretRef to a secret with username and password."
	fi
	cifsUsername="$(base64 --decode <<< "$cifsUsernameBase64" 2>/dev/null)"
	if [[ $? -ne 0 ]] ; then
		errorExit "cifs mount: username secret is not base64 encoded."
	fi
	cifsPassword="$(base64 --decode <<< "$cifsPasswordBase64" 2>/dev/null)"
	if [[ $? -ne 0 ]] ; then
		errorExit "cifs mount: password secret is not base64 encoded."
	fi
	if ! mkdir -p "$mountPoint" > /dev/null 2>&1 ; then
		errorExit "cifs mount: failed to create mount directory: '$mountPoint'"
	fi
	if [[ $(mountpoint "$mountPoint") = *"is a mountpoint"* ]] ; then
		errorExit "cifs mount: there is already a filesystem mounted under the mount directory: '$mountPoint'"
	fi
	if [[ ! -z $(ls -A "$mountPoint" 2>/dev/null) ]] ; then
		errorExit "cifs mount: mount directory is not an empty directory: '$mountPoint'"
	fi

	export PASSWD="$cifsPassword"
	result=$(mount -t cifs "$networkPath" "$mountPoint" -o "username=$cifsUsername,$mountOptions" 2>&1)
	if [[ $? -ne 0 ]] ; then
		errorExit "cifs mount: failed to mount the network path: $result"
	fi
	echo '{ "status": "Success" }'
	exit 0
}

doUnmount() {
	if [[ -z ${1:-} ]] ; then
		errorExit "cifs unmount: syntax error. usage: cifs unmount <mount dir>"
	fi
	mountPoint="$1"
	if [[ $(mountpoint "$mountPoint") != *"is a mountpoint"* ]] ; then
		errorExit "cifs unmount: no filesystem mounted under directory: '$mountPoint'"
	fi
	result=$(umount "$mountPoint" 2>&1)
	if [[ $? -ne 0 ]] ; then
		errorExit "cifs unmount: failed to unmount the network path: $result"
	fi
	echo '{ "status": "Success" }'
	exit 0
}

not_supported() {
	echo '{ "status": "Not supported" }'
	exit 1
}

command=${1:-}
if [[ -n $command ]]; then
	shift
fi

case "$command" in
	init)
		init "$@"
		;;
	mount)
		doMount "$@"
		;;
	unmount)
		doUnmount "$@"
		;;
	*)
		not_supported "$@"
		;;
esac
```


Install follwoing executables before setting the flex volume

```shell
sudo apt-get install cifs-utils -y
sudo apt-get install jq -y
sudo apt-get install util-linux -y
sudo apt-get install coreutils -y
```

Now run follwoing commands:

```shell
VOLUME_PLUGIN_DIR="/usr/libexec/kubernetes/kubelet-plugins/volume/exec"
mkdir -p "$VOLUME_PLUGIN_DIR/fstab~cifs"
cd "$VOLUME_PLUGIN_DIR/fstab~cifs"
// create the above shell file here with name cifs
chmod 755 cifs
```


Now set up SAMBA share using follwoing steps "https://help.ubuntu.com/community/How%20to%20Create%20a%20Network%20Share%20Via%20Samba%20Via%20CLI%20%28Command-line%20interface/Linux%20Terminal%29%20-%20Uncomplicated,%20Simple%20and%20Brief%20Way!"


Now create a secret.yaml file, this secret will be used while mounting the path:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cifs-secret
  namespace: default
type: fstab/cifs
data:
  username: '<base64 encoded user name of samba share>'
  password: '<base64 encoded password of samba share>'
```

Pod.yaml:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - name: busybox
    image: busybox
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: test
      mountPath: /data
  volumes:
  - name: test
    flexVolume:
      driver: "fstab/cifs"
      fsType: "cifs"
      secretRef:
        name: "cifs-secret"
      options:
        networkPath: "//<server_ip>/<sharename/sharefoldername>"
        mountOptions: "dir_mode=0755,file_mode=0644,noperm"
```

Apply both the file to kubernetes.

Now to check if this works or not:

```shell
kubectl exec -ti busybox /bin/sh
ls data
// create a new file in data, and then check in your actual samba server folder the same file
```

## k8s install and use CSI (Container Storage Interface)

For Samba share, we will use a csi driver created by kubernetes comunity (https://github.com/kubernetes-csi/csi-driver-smb)

### Install csi-driver-smb

```shell
curl -skSL https://raw.githubusercontent.com/kubernetes-csi/csi-driver-smb/v1.1.0/deploy/install-driver.sh | bash -s v1.1.0 --
```

After installation check pod status:

```shell
kubectl -n kube-system get pod -o wide --watch -l app=csi-smb-controller
kubectl -n kube-system get pod -o wide --watch -l app=csi-smb-node
```

### Access existing samba share with PVC

### Create PV:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-smb
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  mountOptions:
    - dir_mode=0777
    - file_mode=0777
    - vers=3.0
  csi:
    driver: smb.csi.k8s.io
    readOnly: false
    volumeHandle: unique-volumeid  # make sure it's a unique id in the cluster
    volumeAttributes:
      source: "//192.168.13.128/sharedpath" # smb server path
    nodeStageSecretRef:
      name: cifs-secret # secret name that hase username and password of samba share
      namespace: default
```

### Create PVC:

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-smb
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  volumeName: pv-smb
  storageClassName: ""
```

### Create Pod with PVC:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busyboxwithpvc
  namespace: default
spec:
  containers:
  - name: busyboxwithpvc
    image: busybox
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: test
      mountPath: /data
  volumes:
  - name: test
    persistentVolumeClaim:
      claimName: pvc-smb

```

To verify if share is working use follwoing command:

```shell
kubectl exec -ti busyboxwithpvc /bin/sh
ls data
```

### NOTE: In case of error after creating pod, check logs in smb container of csi-smb-node deployments:

```shell
kubectl logs csi-smb-node-8kwn8 -n kube-system -c smb
```

