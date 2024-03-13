## OpenShift Playground (Tekton CI/CD, Monitoring, etc.)

OpenShift builds atop its Kubernetes core to add features and the components that
support them. Some of its original developers called Kubernetes “a platform for
building platforms.” OpenShift took them up on it. It provides the automation and
resilience of modern infrastructure while letting you stay focused on your application
code. Kubernetes in OpenShift is like the Linux kernel in a Linux distribution.

<img src="pictures/OpenShift-oround-a-K8s-core.png?raw=true" width="800">

Note:  We will use CRC in this playground and setting up CodeReady Containers (CRC) on a Remote Server (Ubuntu 22.04) and Remote Access to CRC from Laptop (remote OpenShift4 development environment). 

Pre: RHN account (free) 

#### 1. Setup OpenShift CRC on remote host (Ubuntu 22.04 LTS, Server IP: 192.168.1.99 devops)
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
$ crc config set memory 14336
$ crc config set consent-telemetry no
$ crc config set disk-size 100
$ crc config set enable-cluster-monitoring true
$ crc config set pull-secret-file ~/OPENSHIFT/pull-secret.txt
Successfully configured pull-secret-file to /home/davar/OPENSHIFT/pull-secret.txt
$ crc config view
- consent-telemetry                     : no
- disk-size                             : 100
- enable-cluster-monitoring             : true
- memory                                : 14336
- pull-secret-file                      : /home/davar/OPENSHIFT-CRC/pull-secrets.txt

### Deploy CodeReady Containers virtual machine
$ crc setup
$ crc start
INFO Using bundle path /home/davar/.crc/cache/crc_libvirt_4.14.12_amd64.crcbundle 
INFO Checking if running as non-root              
INFO Checking if running inside WSL2              
INFO Checking if crc-admin-helper executable is cached 
INFO Checking if running on a supported CPU architecture 
INFO Checking if crc executable symlink exists    
INFO Checking minimum RAM requirements            
INFO Checking if Virtualization is enabled        
INFO Checking if KVM is enabled                   
INFO Checking if libvirt is installed             
INFO Checking if user is part of libvirt group    
INFO Checking if active user/process is currently part of the libvirt group 
INFO Checking if libvirt daemon is running        
INFO Checking if a supported libvirt version is installed 
INFO Checking if crc-driver-libvirt is installed  
INFO Checking crc daemon systemd socket units     
INFO Checking if AppArmor is configured           
INFO Checking if systemd-networkd is running      
INFO Checking if NetworkManager is installed      
INFO Checking if NetworkManager service is running 
INFO Checking if dnsmasq configurations file exist for NetworkManager 
INFO Checking if the systemd-resolved service is running 
INFO Checking if /etc/NetworkManager/dispatcher.d/99-crc.sh exists INFO Checking if libvirt 'crc' network is available 
INFO Checking if libvirt 'crc' network is active  
INFO Loading bundle: crc_libvirt_4.14.12_amd64... 
INFO Starting CRC VM for openshift 4.14.12...     
INFO CRC instance is running with IP 192.168.130.11 
INFO CRC VM is running                            
INFO Updating authorized keys...                  
INFO Configuring shared directories               
INFO Check internal and public DNS query...       
INFO Check DNS query from host...                 
WARN Wildcard DNS resolution for apps-crc.testing does not appear to be working 
INFO Verifying validity of the kubelet certificates... 
INFO Starting kubelet service                     
INFO Waiting for kube-apiserver availability... [takes around 2min] 
INFO Waiting until the user's pull secret is written to the instance disk... 
INFO Enabling cluster monitoring operator...      
INFO Starting openshift instance... [waiting for the cluster to stabilize] 
INFO Operator network is progressing              
INFO 2 operators are progressing: ingress, network 
INFO Operator ingress is progressing              
INFO Operator ingress is progressing              
INFO 2 operators are progressing: image-registry, ingress 
INFO Operator ingress is progressing              
INFO Operator ingress is progressing              
INFO All operators are available. Ensuring stability... 
INFO Operators are stable (2/3)...                
INFO Operators are stable (3/3)...                
INFO Adding crc-admin and crc-developer contexts to kubeconfig... 
Started the OpenShift cluster.

The server is accessible via web console at:
  https://console-openshift-console.apps-crc.testing

Log in as administrator:
  Username: kubeadmin
  Password: kTJ5D-VrUDY-NEuuV-kwG5J

Log in as user:
  Username: developer
  Password: developer

Use the 'oc' command line interface:
  $ eval $(crc oc-env)
  $ oc login -u developer https://api.crc.testing:6443

### Check CRC version and credentials
$ crc version
CRC version: 2.33.0+c43b17
OpenShift version: 4.14.12
Podman version: 4.4.4

$  crc console --credentials
To login as a regular user, run 'oc login -u developer -p developer https://api.crc.testing:6443'.
To login as an admin, run 'oc login -u kubeadmin -p kTJ5D-VrUDY-NEuuV-kwG5J https://api.crc.testing:6443'

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
#### 2.Setup CRC access (oc CLI && OC Console) on the Laptop

```
Add line /etc/hosts

192.168.1.99 devops console-openshift-console.apps-crc.testing oauth-openshift.apps-crc.testing

Browser: https://console-openshift-console.apps-crc.testing/dashboards

Install oc CLI on laptop && oc login

$ oc login -u kubeadmin -p kTJ5D-VrUDY-NEuuV-kwG5J https://console-openshift-console.apps-crc.testing:6443
Login successful.

You have access to 67 projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "default"

Or: $ oc login -u kubeadmin -p kTJ5D-VrUDY-NEuuV-kwG5J https://devops:6443

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
etc.

```

### 3.Tekton CI/CD Pipelines


Install Red Hat OpenShift Pipelines via OpenShift Web Console && downlad tkt CLI 

Note: Red Hat OpenShift Pipelines is based on the Tekton CI/CD project.

#### Create new project

```
$ oc new-project arcade

Note: To switch to a different project, you can use the following command:
$ oc project default
To switch back to the arcade project, run the following command accordingly:
$ oc project arcade
```

#### Run Tasks

```
• Task 1
— Step 1: Clone Git repository
— Step 2: Run unit tests
• Task 2
— Step 1: Configure project
— Step 2: Build application
— Step 3: Deploy and expose the application


$ cd highscore/ci

$ oc apply -f task-unit.yaml
$ tkn task start verify-unit-tests --showlog


$ oc apply -f rbac.yaml
Note: For the dynamic generation of a hostname for the route, the following RBAC
resources need to be created in the cluster, as the default pipeline user doesn’t have
permission to read the ingress resource:

$ oc apply -f task-deploy.yaml
$ tkn task start build-deploy --showlog

Note: The second task (build-deploy) mostly represents what you did manually:
$ oc new-project arcade
$ oc new-build https://github.com/adavarski/OpenShift-CRC-Playground --context-dir=highscore \
--name=highscore
$ oc start-build highscore
$ oc create deployment highscore \
--image=image-registry.openshift-image-registry.svc:5000/arcade/highscore
$ oc expose deployment highscore
$ oc expose service highscore \
--hostname=arcade.apps-crc.testing --path=/highscore
```
### Composing the pipeline and Run Pipeline

However, since the idea of a pipeline is that it will run more than once, you need to
make sure that the pipeline is idempotent. That means, independent of how often the
pipeline is started, the OpenShift cluster will, in the end, get the latest version of the
application deployed. When the pipeline runs the first time, it’ll create the project and
deploy the resources. If it runs the second time, those resources already exist. If it
runs again and everything is up-to-date already, the pipeline doesn’t change anything.
To achieve this, you can use the declarative way of defining the resources the pipeline
```
### Run Pipeline

$ oc apply -f pipeline.yaml
pipeline.tekton.dev/build-pipeline created

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

$ tkn pipelines list
NAME             AGE              LAST RUN                   STARTED          DURATION   STATUS
build-pipeline   18 minutes ago   build-pipeline-run-z6ldd   18 minutes ago   2m17s      Succeeded

```

Add arcade to /etc/hosts and test:
```
192.168.1.99 devops console-openshift-console.apps-crc.testing oauth-openshift.apps-crc.testing arcade.apps-crc.testing

$ curl arcade.apps-crc.testing/highscore
<html><body><h1>Highscores</h1></html>
```

Test Tekton EvenListener : 
```
$ oc apply -f eventlistener.yaml
eventlistener.triggers.tekton.dev/build-pipeline-listener created

List services to see the service that is created by Tekton:
$ oc get services
NAME                         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)             AGE
el-build-pipeline-listener   ClusterIP   10.217.5.6    <none>        8080/TCP,9000/TCP   160m
highscore                    ClusterIP   10.217.4.50   <none>        8080/TCP            3h23m

To test the EventListener, create a curl pod and send an empty JSON via POST to the
service:
$ oc run curl --image=curlimages/curl --command sleep 30h
pod/curl created
$ oc exec curl -- curl -s el-build-pipeline-listener:8080 -X POST --data '{}'
{"eventListener":"build-pipeline-listener","namespace":"arcade",
"eventListenerUID":"aecac3b4-a865-44df-92f0-4ac470a4bae4",
"eventID":"a1bb9392-5f58-47fb-bdfa-c68736fd690c"}
```

Night builds (via cronjob):
```
$ oc apply -f cronjob.yaml
cronjob.batch/highscore-nightly-build created
```
TBD: Pipeline triggered by GitHub push event (webhook), Note: Needs OC public IP.

### OpenShift Gitops 

OpenShift GitOps is based on Argo CD. To install OpenShift GitOps using the OpenShift Console, visit the OperatorHub
section in the OpenShift console, search for OpenShift GitOps and click Install
```
$ oc extract secret/openshift-gitops-cluster -n openshift-gitops --to=-
# admin.password
6eJ9jmwT4vcFDZABPgYWNCk73sEutyqd
```
$ oc project openshift-gitops
Now using project "openshift-gitops" on server "https://console-openshift-console.apps-crc.testing:6443".
$ oc get route
NAME                      HOST/PORT                                                   PATH   SERVICES                  PORT    TERMINATION            WILDCARD
kam                       kam-openshift-gitops.apps-crc.testing                              kam                       8443    passthrough/None       None
openshift-gitops-server   openshift-gitops-server-openshift-gitops.apps-crc.testing          openshift-gitops-server   https   passthrough/Redirect   None

Add "192.168.1.99 openshift-gitops-server-openshift-gitops.apps-crc.testing" to /etc/hosts and Browser Access ArgoCD Web UI: https://openshift-gitops-server-openshift-gitops.apps-crc.testing/applications (user: admin; password: 6eJ9jmwT4vcFDZABPgYWNCk73sEutyqd)


### Deploying Grafana on Openshift 4

OpenShift users want access to a Grafana interface in order to build custom dashboards for their cluster and application workloads. The Grafana that shipped with OpenShift was read-only and has been deprecated in OpenShift 4.11 and removed in OpenShift 4.12 [REF](https://issues.redhat.com/browse/MON-1591?focusedCommentId=19239654&page=com.atlassian.jira.plugin.system.issuetabpanels%3Acomment-tabpanel#comment-19239654).

Since OpenShift uses Prometheus for both Cluster and User Workload metrics, its fairly straight forward to deploy a Grafana instance using the Grafana Operator and then view those cluster metrics and create custom Dashboards.

Grafana Operator
Create a namespace for the Grafana Operator to be installed in

```
oc new-project grafana-operator
```

Deploy the Grafana Operator
```
cat << EOF | oc create -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  generateName: grafana-operator-
  namespace: grafana-operator
spec:
  targetNamespaces:
  - grafana-operator
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  generateName: grafana-operator-
  namespace: grafana-operator
spec:
  channel: v4
  name: grafana-operator
  installPlanApproval: Automatic
  source: community-operators
  sourceNamespace: openshift-marketplace
EOF
```
Wait for the Operator to be ready

```
oc -n grafana-operator rollout status \
  deployment grafana-operator-controller-manager
```
Deploy Grafana
Add the Managed OpenShift Black Belt helm repo
```
helm repo add mobb https://rh-mobb.github.io/helm-charts/
```
Deploy Grafana
```
helm upgrade --install -n grafana-operator \
  grafana mobb/grafana-cr --set "basicAuthPassword=myPassword"
```
Create Prometheus DataSource
Give the Grafana service account the cluster-monitoring-view role
```
oc adm policy add-cluster-role-to-user \
  cluster-monitoring-view -z grafana-serviceaccount
```
Get a bearer token for the grafana service account
```
BEARER_TOKEN=$(oc create token grafana-serviceaccount)
```
Deploy the prometheus data source
```
cat << EOF | oc apply -f -
apiVersion: integreatly.org/v1alpha1
kind: GrafanaDataSource
metadata:
  name: prometheus-grafanadatasource
spec:
  datasources:
    - access: proxy
      editable: true
      isDefault: true
      jsonData:
        httpHeaderName1: 'Authorization'
        timeInterval: 5s
        tlsSkipVerify: true
      name: Prometheus
      secureJsonData:
        httpHeaderValue1: 'Bearer ${BEARER_TOKEN}'
      type: prometheus
      url: 'https://thanos-querier.openshift-monitoring.svc.cluster.local:9091'
  name: prometheus-grafanadatasource.yaml
EOF
```
Deploy a Dashboard
```
oc apply -f https://raw.githubusercontent.com/kevchu3/openshift4-grafana/master/dashboards/crds/cluster_metrics.ocp412.grafanadashboard.yaml
```
Get the URL for Grafana
```
oc get route grafana-route -o jsonpath='{"https://"}{.spec.host}{"\n"}'
```
Add "192.168.1.99 grafana-route-grafana-operator.apps-crc.testing" to /etc/hosts and browse to the URL from above, log in via your OCP credentials

Click Dashboards -> Manage -> grafana-operator -> Cluster Metrics
