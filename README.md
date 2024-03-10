### Setting up CodeReady Containers (CRC) on a Remote Server (Ubuntu 22.04) and Remote Access to CRC (remote OpenShift4 development environment) from Laptop

Pre: RHN account (free) 

#### 1. Setup OpenShift CRC on remote host (Ubuntu 22.04 LTS, Server IP: 192.168.1.99)
```

### Setup sudo

$ echo "$USER ALL=(ALL) NOPASSWD:ALL"|sudo tee -a /etc/sudoers

### Install packages

$ sudo apt install qemu-kvm libvirt-daemon libvirt-daemon-system network-manager haproxy

### Download CRC and pull-secret in OPENSIFT-CRC directory form RHN:
$ mkdir OPENSHIFT-CRC && cd OPENSHIFT-CRC/
$ wget https://mirror.openshift.com/pub/openshift-v4/clients/crc/2.33.0/crc-linux-amd64.tar.xz
$ tar -xJf ./crc-linux-amd64.tar.xz

### Place the binaries in your $PATH using .bash_profile
$ echo "export PATH=/home/davar/OPENSHIFT/crc-linux-2.33.0-amd64:/home/davar/.crc/bin/oc:$PATH" >> ~/.bash_profile
$ source ~/.bash_profile (or relogin)
### Place the binaries in your $PATH using .bashrc
$ vim ~/.bashrc
export PATH="~/.crc/bin:$PATH"
eval $(crc oc-env)
$ source ~/.bashrc 

### Setup CRC:
$ crc config set memory 12288
$ crc config set consent-telemetry no
$ crc config set disk-size 100
$ crc config set pull-secret-file ~/OPENSHIFT/pull-secret.txt
Successfully configured pull-secret-file to /home/davar/OPENSHIFT/pull-secret.txt
$ crc config view
- consent-telemetry                     : no
- disk-size                             : 100
- memory                                : 12288
- pull-secret-file                      : /home/davar/OPENSHIFT-CRC/pull-secrets.txt

### Deploy CodeReady Containers virtual machine
$ crc setup
$ crc start

### Check CRC version and credentials
$ crc version
CRC version: 2.33.0+c43b17
OpenShift version: 4.14.12
Podman version: 4.4.4

$  crc console --credentials
To login as a regular user, run 'oc login -u developer -p developer https://api.crc.testing:6443'.
To login as an admin, run 'oc login -u kubeadmin -p VjVIZ-rDfq3-S5Ip2-4jiLU https://api.crc.testing:6443'

### Check if CodeReady Containers is working
$ oc login -u kubeadmin -p kTJ5D-VrUDY-NEuuV-kwG5J https://api.crc.testing:6443
$ oc get nodes

### Setup HAProxy for remote CRC access 
sudo apt install haproxy -y
sudo cp /etc/haproxy/haproxy.cfg{,.bak}

CRC_IP=$(crc ip)
sudo tee /etc/haproxy/haproxy.cfg &>/dev/null <<EOF
global
    log /dev/log local0

defaults
    balance roundrobin
    log global
    maxconn 100
    mode tcp
    timeout connect 5s
    timeout client 500s
    timeout server 500s

listen apps
    bind 0.0.0.0:80
    server crcvm $CRC_IP:80 check

listen apps_ssl
    bind 0.0.0.0:443
    server crcvm $CRC_IP:443 check

listen api
    bind 0.0.0.0:6443
    server crcvm $CRC_IP:6443 check
EOF

sudo systemctl restart haproxy
sudo systemctl enable haproxy
```
#### 2.Setup On the Laptop

```
Add line /etc/hosts

192.168.1.99 devops console-openshift-console.apps-crc.testing oauth-openshift.apps-crc.testing

Browser: https://console-openshift-console.apps-crc.testing/dashboards

Install oc CLI on laptop && oc login

$ oc login -u kubeadmin -p kTJ5D-VrUDY-NEuuV-kwG5J https://console-openshift-console.apps-crc.testing:6443
Login successful.

You have access to 67 projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "default"

Or: $ oc login -u kubeadmin -p VjVIZ-rDfq3-S5Ip2-4jiLU https://devops:6443

$ cat ~/.kube/config ($ oc config view)
apiVersion: v1
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: https://console-openshift-console.apps-crc.testing:6443
  name: console-openshift-console-apps-crc-testing:6443
contexts:
- context:
    cluster: console-openshift-console-apps-crc-testing:6443
    namespace: default
    user: kubeadmin/console-openshift-console-apps-crc-testing:6443
  name: default/console-openshift-console-apps-crc-testing:6443/kubeadmin
current-context: default/console-openshift-console-apps-crc-testing:6443/kubeadmin
kind: Config
preferences: {}
users:
- name: kubeadmin/console-openshift-console-apps-crc-testing:6443
  user:
    token: sha256~ht4d-DeKbHofi0uSCEuQ3WwoQmdS73s_o9AyJL7P3IE


$ oc get po


```

### Tekton CI/CD 


Install Red Hat OpenShift Pipelines via OC Console && downlad tkt CLI 

```
### Create new project

$ oc new-project arcade
```
Note: To switch to a different project, you can use the following command:
```
$ oc project default
To switch back to the arcade project, run the following command accordingly:
$ oc project arcade
```

### Run Pipeline
Note: For the dynamic generation of a hostname for the route, the following RBAC
resources need to be created in the cluster, as the default pipeline user doesn’t have
permission to read the ingress resource:

$ oc apply -f rbac.yaml

$ tkn pipeline start build-pipeline --showlog
```
$ tkn pipeline start build-pipeline --showlog
PipelineRun started: build-pipeline-run-fstdf
Waiting for logs to be available...
[verify-unit-tests : clone] Cloning into '/workspace/src'...

[verify-unit-tests : test-unit] go: downloading github.com/sdomino/scribble v0.0.0-20200707180004-3cc68461d505
[verify-unit-tests : test-unit] go: downloading github.com/jcelliott/lumber v0.0.0-20160324203708-dd349441af25
[verify-unit-tests : test-unit] ok  	github.com/NautiluX/s3e/highscore	0.004s

[build-deploy : clone] Cloning into '/workspace/src'...

[build-deploy : configure-project] imagestream.image.openshift.io/highscore unchanged
[build-deploy : configure-project] imagestream.image.openshift.io/alpine unchanged
[build-deploy : configure-project] buildconfig.build.openshift.io/highscore unchanged

[build-deploy : build] build.build.openshift.io/highscore-5 started
[build-deploy : build] Cloning "https://github.com/NautiluX/s3e" ...
[build-deploy : build] 	Commit:	272c3ee6cbcb3cf8ba8ad3072f45f34a5ee409ea (add cert-manager files)
[build-deploy : build] 	Author:	Manuel Dewald <mdewald@redhat.com>
[build-deploy : build] 	Date:	Mon Jul 11 16:39:45 2022 +0200
[build-deploy : build] Replaced Dockerfile FROM image alpine:latest
[build-deploy : build] time="2024-03-10T10:34:05Z" level=info msg="Not using native diff for overlay, this may cause degraded performance for building images: kernel has CONFIG_OVERLAY_FS_REDIRECT_DIR enabled"
[build-deploy : build] I0310 10:34:05.429846       1 defaults.go:112] Defaulting to storage driver "overlay" with options [mountopt=metacopy=on].
[build-deploy : build] Caching blobs under "/var/cache/blobs".
[build-deploy : build] 
[build-deploy : build] Pulling image alpine@sha256:6457d53fb065d6f250e1504b9bc42d5b6c65941d57532c072d929dd0628977d0 ...
[build-deploy : build] Resolving "alpine" using unqualified-search registries (/etc/containers/registries.conf)
[build-deploy : build] Trying to pull registry.redhat.io/alpine@sha256:6457d53fb065d6f250e1504b9bc42d5b6c65941d57532c072d929dd0628977d0...
[build-deploy : build] Trying to pull registry.access.redhat.com/alpine@sha256:6457d53fb065d6f250e1504b9bc42d5b6c65941d57532c072d929dd0628977d0...
[build-deploy : build] Trying to pull quay.io/alpine@sha256:6457d53fb065d6f250e1504b9bc42d5b6c65941d57532c072d929dd0628977d0...
[build-deploy : build] Trying to pull docker.io/library/alpine@sha256:6457d53fb065d6f250e1504b9bc42d5b6c65941d57532c072d929dd0628977d0...
[build-deploy : build] Getting image source signatures
[build-deploy : build] Copying blob sha256:4abcf20661432fb2d719aaf90656f55c287f8ca915dc1c92ec14ff61e67fbaf8
[build-deploy : build] Copying config sha256:05455a08881ea9cf0e752bc48e61bbd71a34c029bb13df01e40e3e70e0d007bd
[build-deploy : build] Writing manifest to image destination
[build-deploy : build] 
[build-deploy : build] Pulling image golang:1.14 ...
[build-deploy : build] Resolving "golang" using unqualified-search registries (/etc/containers/registries.conf)
[build-deploy : build] Trying to pull registry.redhat.io/golang:1.14...
[build-deploy : build] Trying to pull registry.access.redhat.com/golang:1.14...
[build-deploy : build] Trying to pull quay.io/golang:1.14...
[build-deploy : build] Trying to pull docker.io/library/golang:1.14...
[build-deploy : build] Getting image source signatures
[build-deploy : build] Copying blob sha256:48bbd1746d63c372e12f884178053851d87f3ea4b415f3d5e7d4eceb9aab6baf
[build-deploy : build] Copying blob sha256:7467d1831b6947c294d92ee957902c3cd448b17c5ac2103ca5e79d15afb317c3
[build-deploy : build] Copying blob sha256:1517911a35d7939f446084c1d4c31afc552678e81b3450c2af998b57f72099c2
[build-deploy : build] Copying blob sha256:0ecb575e629cd60aa802266a3bc6847dcf4073aa2a6d7d43f717dd61e7b90e0b
[build-deploy : build] Copying blob sha256:feab2c490a3cea21cc051ff29c33cc9857418edfa1be9966124b18abe1d5ae16
[build-deploy : build] Copying blob sha256:f15a0f46f8c38f4ca7daecf160ba9cdb3ddeafda769e2741e179851cfaa14eec
[build-deploy : build] Copying blob sha256:944903612fdd2364b4647cf3c231c41103d1fd378add43fc85f6931f1d6a8276
[build-deploy : build] Copying config sha256:21a5635903d69da3c3d928ed429e3610eecdf878cbd763aabf5db9693016ffbb
[build-deploy : build] Writing manifest to image destination
[build-deploy : build] Adding transient rw bind mount for /run/secrets/rhsm
[build-deploy : build] [1/2] STEP 1/4: FROM golang:1.14 AS build
[build-deploy : build] [1/2] STEP 2/4: COPY . /src
[build-deploy : build] --> 9e8a4d21466e
[build-deploy : build] [1/2] STEP 3/4: WORKDIR /src
[build-deploy : build] --> 11ba924f2ba5
[build-deploy : build] [1/2] STEP 4/4: RUN CGO_ENABLED=0 GOOS=linux go build
[build-deploy : build] go: downloading github.com/sdomino/scribble v0.0.0-20200707180004-3cc68461d505
[build-deploy : build] go: downloading github.com/jcelliott/lumber v0.0.0-20160324203708-dd349441af25
[build-deploy : build] --> 7306342b44fc
[build-deploy : build] [2/2] STEP 1/11: FROM alpine@sha256:6457d53fb065d6f250e1504b9bc42d5b6c65941d57532c072d929dd0628977d0
[build-deploy : build] [2/2] STEP 2/11: RUN apk --no-cache add ca-certificates
[build-deploy : build] fetch https://dl-cdn.alpinelinux.org/alpine/v3.19/main/x86_64/APKINDEX.tar.gz
[build-deploy : build] fetch https://dl-cdn.alpinelinux.org/alpine/v3.19/community/x86_64/APKINDEX.tar.gz
[build-deploy : build] (1/1) Installing ca-certificates (20230506-r0)
[build-deploy : build] Executing busybox-1.36.1-r15.trigger
[build-deploy : build] Executing ca-certificates-20230506-r0.trigger
[build-deploy : build] OK: 8 MiB in 16 packages
[build-deploy : build] --> 10ecf0182dc8
[build-deploy : build] [2/2] STEP 3/11: WORKDIR /app
[build-deploy : build] --> 313cd86f9fd3
[build-deploy : build] [2/2] STEP 4/11: RUN mkdir -p /app/db/GameScores
[build-deploy : build] --> f47ea6999bd9
[build-deploy : build] [2/2] STEP 5/11: RUN chgrp -R 0 /app/db && chmod -R g=u /app/db
[build-deploy : build] --> c39753715096
[build-deploy : build] [2/2] STEP 6/11: COPY --from=build /src/highscore .
[build-deploy : build] --> c41991f873a5
[build-deploy : build] [2/2] STEP 7/11: RUN ls /app
[build-deploy : build] db
[build-deploy : build] highscore
[build-deploy : build] --> 710e69947ed2
[build-deploy : build] [2/2] STEP 8/11: EXPOSE 8080
[build-deploy : build] --> fe6443b5662b
[build-deploy : build] [2/2] STEP 9/11: CMD ["/app/highscore"]
[build-deploy : build] --> 25bbac65d0c5
[build-deploy : build] [2/2] STEP 10/11: ENV "OPENSHIFT_BUILD_NAME"="highscore-5" "OPENSHIFT_BUILD_NAMESPACE"="arcade" "OPENSHIFT_BUILD_SOURCE"="https://github.com/NautiluX/s3e" "OPENSHIFT_BUILD_COMMIT"="272c3ee6cbcb3cf8ba8ad3072f45f34a5ee409ea"
[build-deploy : build] --> 0a70ed33d9d6
[build-deploy : build] [2/2] STEP 11/11: LABEL "io.openshift.build.commit.author"="Manuel Dewald <mdewald@redhat.com>" "io.openshift.build.commit.date"="Mon Jul 11 16:39:45 2022 +0200" "io.openshift.build.commit.id"="272c3ee6cbcb3cf8ba8ad3072f45f34a5ee409ea" "io.openshift.build.commit.message"="add cert-manager files" "io.openshift.build.commit.ref"="main" "io.openshift.build.name"="highscore-5" "io.openshift.build.namespace"="arcade" "io.openshift.build.source-context-dir"="highscore" "io.openshift.build.source-location"="https://github.com/NautiluX/s3e"
[build-deploy : build] [2/2] COMMIT temp.builder.openshift.io/arcade/highscore-5:66a29004
[build-deploy : build] --> 2e2c99397f94
[build-deploy : build] Successfully tagged temp.builder.openshift.io/arcade/highscore-5:66a29004
[build-deploy : build] 2e2c99397f940cdf6a1e4fb266c466fdaee85064550729f5d2ca50bd6af6241d
[build-deploy : build] 
[build-deploy : build] Pushing image image-registry.openshift-image-registry.svc:5000/arcade/highscore:latest ...
[build-deploy : build] Getting image source signatures
[build-deploy : build] Copying blob sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef
[build-deploy : build] Copying blob sha256:e32cfe9b09d8e94a798ecd26512287f35cee6e0ec038271b98ffbdb53a230f34
[build-deploy : build] Copying blob sha256:4ea0092257d4ad5f2deda5fb9415fa569bbc34c6161052d3d8bf6cbfa6aa6b12
[build-deploy : build] Copying blob sha256:0beb17515d051736b1ca139107d67466b663e9f0c6bade4565b1e3940e7a2693
[build-deploy : build] Copying blob sha256:c76dd42eaab6801bef3fa540315c53f862868612a39a8e402691c8733084fe99
[build-deploy : build] Copying blob sha256:4abcf20661432fb2d719aaf90656f55c287f8ca915dc1c92ec14ff61e67fbaf8
[build-deploy : build] Copying config sha256:2e2c99397f940cdf6a1e4fb266c466fdaee85064550729f5d2ca50bd6af6241d
[build-deploy : build] Writing manifest to image destination
[build-deploy : build] Successfully pushed image-registry.openshift-image-registry.svc:5000/arcade/highscore@sha256:2e4d0e8994ceb8309cd044d341afe9e690632fb91ac1519b0c6245392a878e35
[build-deploy : build] Push successful

[build-deploy : deploy] + cd /workspace/src/highscore/ci
[build-deploy : deploy] + oc apply -f deployment.yaml
[build-deploy : deploy] deployment.apps/highscore configured
[build-deploy : deploy] ++ date +%Y-%m-%d-%H%M%S
[build-deploy : deploy] + BUILDDATE=2024-03-10-103517
[build-deploy : deploy] + oc patch deployment highscore -p '{"spec":{"template":{"metadata":{"labels":{"last-build":"2024-03-10-103517"}}}}}'
[build-deploy : deploy] deployment.apps/highscore patched
[build-deploy : deploy] + oc apply -f service.yaml
[build-deploy : deploy] service/highscore unchanged
[build-deploy : deploy] ++ oc get ingresses.config.openshift.io cluster -o 'jsonpath={.spec.domain}'
[build-deploy : deploy] + HOSTNAME=arcade.apps-crc.testing
[build-deploy : deploy] + oc apply -f /dev/fd/63
[build-deploy : deploy] ++ cat
[build-deploy : deploy] route.route.openshift.io/highscore unchanged
```

Add arcade to /etc/hosts
```
192.168.1.99 devops console-openshift-console.apps-crc.testing oauth-openshift.apps-crc.testing arcade.apps-crc.testing
```

Test : 
```
$ curl arcade.apps-crc.testing/highscore
<html><body><h1>Highscores</h1></html>
```
