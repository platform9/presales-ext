### This article highlights steps to install [Private Cloud Director](https://platform9.com/private-cloud-director/) in an onprem online mode.
#### Refer to [Pre-requisites](https://platform9.com/docs/private-cloud-director/private-cloud-director/pre-requisites) for Host Hardware, Networking and Storage Requirements.

### Prep / Configuration (For all management plane nodes)
#### Install cgroup-tools:
```
apt-get update -y && apt-get install cgroup-tools -y
```
#### Update OpenSSL Version to 3.0.7 for Ubuntu 22.04:
```
curl --user-agent "<REDACTED>" https://pf9-airctl.s3-accelerate.amazonaws.com/openssl-smcp-ubuntu/openssl_3.0.7-1_amd64.deb --output openssl_3.0.7-1_amd64.deb
sudo dpkg -i openssl_3.0.7-1_amd64.deb
cd /etc/ld.so.conf.d/
echo "/usr/local/ssl/lib64" > openssl-3.0.7.conf
ldconfig -v
sed -i 's/\(^PATH="[^"]*\)"/\1:\/usr\/local\/ssl\/bin"/' /etc/environment
source /etc/environment
ln -sf /usr/local/ssl/bin/openssl /usr/bin/openssl
cd
```
> The User agent key will need to be requested from Platform9. 
 
##### Verify OpenSSL Version:
```
openssl version
```
---
### Download Platform9 Artifacts & Set Configuration:
#### To be done only on one of management plane nodes
```
curl --user-agent "<REDACTED>" https://pf9-airctl.s3-accelerate.amazonaws.com/latest/index.txt | grep -e airctl -e install-pmo.sh -e nodelet-deb.tar.gz -e nodelet.tar.gz -e pmo-chart.tgz -e options.json | awk '{print "curl -sS --user-agent \"<REDACTED>\" \"https://pf9-airctl.s3-accelerate.amazonaws.com/latest/" $NF "\" -o /root/" $NF}' | bash
```
> The User agent key will need to be requested from Platform9.
```
chmod +x ./install-pmo.sh
```
```
./install-pmo.sh v-5.12.0-3433619
```
```
echo 'export PATH=$PATH:/opt/pf9/airctl' >> ~/.bashrc
source ~/.bashrc
```
#### Run Configure:
```
airctl configure
```
```
# airctl configure
? Select install driver: openstack
? Number of master nodes: 3
? Space separated list of Master Node IPs. Example: 1.1.1.1 2.2.2.2: 172.29.21.157 172.29.20.223 172.29.20.157
? Space separated list of Worker Node IPs(optional). Example: 3.3.3.3 4.4.4.4:
? Network Stack: IPv4
? VIP for management cluster: 172.29.20.175
? Management Plane FQDN: foo.bar.io
? Enter external IP for DU: 172.29.21.1
Generated '/opt/pf9/airctl/conf/nodelet-bootstrap-config.yaml' and '/opt/pf9/airctl/conf/airctl-config.yaml'. Please review and edit any fields as necessary.
```
> [!IMPORTANT] 
> Ensure to select install driver as `openstack`.
> 
> The FQDN you will provide in the above command prompt will be for the `Infra` region which gets populated into `duFqdn` field in `/opt/pf9/airctl/conf/airctl-config.yaml`.

#### Post Command Completion:
> [!IMPORTANT] 
> Populate the field `additionalDuFqdns` with the actual region URLs and edit `duRegion` fields in the configuration file `/opt/pf9/airctl/conf/airctl-config.yaml` in a space separated manner.
> 
> Note that the first region name in `duRegion` field should be `Infra`. The other regions can be named as preferred. Here we have set is as `Region1`.

**Single Region Example:**
```
$ cat /opt/pf9/airctl/conf/airctl-config.yaml | grep -ie duFqdn -ie additionalDuFqdns -ie duRegion
duFqdn: foo.bar.io
additionalDuFqdns: foo-region1.bar.io
duRegion: Infra Region1
```
> [!NOTE]
> In above, `foo.bar.io` corresponds to `Infra` region which will only have Keystone service running.
> 
> The Nova, Neutron, Glance and rest of the services would run on the main regions. From the example, `foo-region1.bar.io` corresponding to `Region1`

If more than 1 region is required, below example can be referred:
```
# cat /opt/pf9/airctl/conf/airctl-config.yaml | grep -ie duFqdn -ie additionalDuFqdns -ie duRegion
duFqdn: foo.bar.io
additionalDuFqdns: foo-region1.bar.io foo-region2.bar.io
duRegion: Infra Region1 Region2
```
> [!WARNING]
> The URL domain should be same for both `duFqdn` and `additionalDuFqdns` i.e. in example `bar.io` as domain is present for the provided `additionalDuFqdns` values.
> Providing `duFqdn` as `foo.bar.io` and `additionalDuFqdns` as `bar-region1.foo.io` is NOT supported.
>
> Also at this point, adding new regions at a later stage is NOT supported. So ensure you configure all required regions before executing next steps.

* For multi-master cluster configuration, add below parameters in `/opt/pf9/airctl/conf/nodelet-bootstrap-config.yaml` file. This file has configuration required to bootstrap the Kubernetes cluster. 
> [!IMPORTANT]
> Change interface name for `masterVipInterface` as per your node. In example we have set it as ens3. The `masterVipVrouterId` value can be any unique number.
```
# cat /opt/pf9/airctl/conf/nodelet-bootstrap-config.yaml | grep -e masterVipEnabled -e masterVipInterface -e masterVipVrouterId
masterVipEnabled: true
masterVipInterface: ens3
masterVipVrouterId: 119
```
> [!NOTE]
> In this file, the parameter `masterIp` has the value populated based on the input provided for prompt `VIP for management cluster`. Ensure that this VIP is different from the input provided for prompt `Enter external IP for DU`.

---
#### [Only if required] To ensure the deployments happens with a proxy:
* Set the required values in file `/opt/pf9/airctl/conf/helm_values/kplane.template.yml`
> Example:
```
# cat /opt/pf9/airctl/conf/helm_values/kplane.template.yml | grep proxy
https_proxy: "http://squid.platform9.horse:3128"
http_proxy: "http://squid.platform9.horse:3128"
no_proxy: "172.29.20.208,172.29.21.80,172.29.21.27,127.0.0.1,10.20.0.0/22,localhost,::1,.svc,.svc.cluster.local,10.21.0.0/16,10.20.0.0/16,.cluster.local,.bar.io,.default.svc"
```
* Also, to ensure containerd honors the proxy values so that management cluster can be created, update proxy values on all management plane nodes:
> Example:
```
# cat /etc/environment
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/usr/local/ssl/bin"
HTTP_PROXY="http://squid.platform9.horse:3128"
https_proxy="http://squid.platform9.horse:3128"
http_proxy="http://squid.platform9.horse:3128"
HTTPS_PROXY="http://squid.platform9.horse:3128"
NO_PROXY="10.149.105.43,127.0.0.1,10.20.0.0/22,localhost,::1,.svc,.svc.cluster.local,10.21.0.0/16,10.20.0.0/16,.cluster.local,.platform9.localnet,.default.svc"
no_proxy="10.149.105.43,127.0.0.1,10.20.0.0/22,localhost,::1,.svc,.svc.cluster.local,10.21.0.0/16,10.20.0.0/16,.cluster.local,.platform9.localnet,.default.svc"
```
```
# cat /etc/systemd/system/containerd.service.d/http-proxy.conf
[Service]
EnvironmentFile=/etc/environment
```
---
### Deployment of Management Cluster

#### Create Nodelet Multi-Master K8s cluster:
```
airctl advanced-ddu create-mgmt --config /opt/pf9/airctl/conf/airctl-config.yaml --verbose
```
> [!WARNING]
> Ensure the nodes to be bootstrapped as Kubernetes nodes have swap disabled on them.

Cluster post-creation:
```
export KUBECONFIG=/etc/pf9/kube.d/kubeconfigs/admin.yaml
kubectl get nodes --kubeconfig='/etc/pf9/kube.d/kubeconfigs/admin.yaml'
```
```
# kubectl get nodes --kubeconfig='/etc/pf9/kube.d/kubeconfigs/admin.yaml'
NAME            STATUS   ROLES    AGE     VERSION
172.29.20.157   Ready    master   4m29s   v1.29.2
172.29.20.223   Ready    master   5m41s   v1.29.2
172.29.21.157   Ready    master   5m42s   v1.29.2
```
> [!NOTE]
> Please refer to `/var/log/pf9/nodelet.log` for cluster creation troubleshooting
---
### Execute install
#### Run airctl init:
```
airctl init --config /opt/pf9/airctl/conf/airctl-config.yaml
```
```
# airctl init --config /opt/pf9/airctl/conf/airctl-config.yaml
extracting helm
```
#### Run airctl start:
```
airctl start --config /opt/pf9/airctl/conf/airctl-config.yaml
```
```
# airctl start --config /opt/pf9/airctl/conf/airctl-config.yaml
 INFO  openstack management plane creation started
 SUCCESS  generating certs and config...                                                                                                                          
 SUCCESS  setting up base infrastructure...                                                                                                                      
▀  starting consul...Secret consul-gossip-encryption-key in namespace default not found, creating new...
 INFO  kplane setup done, creating management plane                                                                                                     
 INFO  starting openstack deployment...                                                                                                                         
 SUCCESS  openstack deployment now complete                                                                                                                       
 INFO  openstack management plane created - the services will take a while to start
```

> [!NOTE]
> It will take up to 45 minutes for all the services to get deployed for Infra and 1 region.

#### Monitoring Installation Progress:
We can monitor the progress of the DU deployment by looking at the logs of the `du-install` pod.
```
export KUBECONFIG=/etc/pf9/kube.d/kubeconfigs/admin.yaml
kubectl get pod -A | grep du-install
kubectl logs -n <ns> <pod name> -f
```
```
# kubectl get pods -n foo-kplane | grep du-install
du-install-foo-bmqqd                        0/1     Completed   0          108m
du-install-foo-region1-f7fdw                0/1     Completed   0          98m
```
> [!NOTE]
> Please refer to airctl-logs/airctl.log for logs in case of any issues with `airctl start` command.

#### To check the status of management plane/regions:
```
airctl status --config /opt/pf9/airctl/conf/airctl-config.yaml
```
```
# airctl status --config /opt/pf9/airctl/conf/airctl-config.yaml
------------- deployment details ---------------
fqdn:                foo.bar.io
cluster:             foo-kplane.bar.io
region:              foo
task state:          ready
-------- region service status ----------
desired services:  22
ready services:    22
------------- deployment details ---------------
fqdn:                foo-region1.bar.io
cluster:             foo-kplane.bar.io
region:              foo-region1
task state:          ready
-------- region service status ----------
desired services:  44
ready services:    44
```

#### Obtaining UI Credentials:
```
airctl get-creds --config /opt/pf9/airctl/conf/airctl-config.yaml
```
```
# airctl get-creds --config /opt/pf9/airctl/conf/airctl-config.yaml
email:     admin@airctl.localnet
password:  qaChMBjgkghucTnv
```
---
> [!IMPORTANT]
> On the local machine **AND** any new host that wants to be authorized to the created management plane, update the `/etc/hosts` file with the external IP and FQDN corresponding to it.
>
> This only applies if customers do not have a working internal DNS that will resolve the management plane FQDN to IP.
```
# cat /etc/hosts | grep foo
<VIP for externalIP>         foo-region1.bar.io
```

---
### Preparing Hosts for Authorization to Management Plane
#### Add Management Plane Certificate to all Hosts:
> [!NOTE]
> Since the DU uses self-signed certificates, the API calls from `pmo-ctl` will fail.
>
> You need to add the DU certificate using the following commands for Ubuntu 22 on all hypervisor hosts:
```
# Export DU_URL
export DU_URL=<>

# Get the DUs self-signed certificate
openssl s_client -showcerts -connect $DU_URL:443 2>/dev/null </dev/null | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > du.crt

# Copy it to trusted store
cp du.crt /usr/local/share/ca-certificates

# Refresh CA certs
sudo update-ca-certificates

# Ensure curl does not complain about the certificate
curl https://$DU_URL
```
```
# export DU_URL=foo-region1.bar.io 

# openssl s_client -showcerts -connect $DU_URL:443 2>/dev/null </dev/null | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > du.crt

# cp du.crt /usr/local/share/ca-certificates

# sudo update-ca-certificates
Updating certificates in /etc/ssl/certs...
rehash: warning: skipping ca-certificates.crt,it does not contain exactly one certificate or CRL
1 added, 0 removed; done.
Running hooks in /etc/ca-certificates/update.d...
done.

# curl https://$DU_URL/keystone/v3
{"version": {"id": "v3.14", "status": "stable", "updated": "2020-04-07T00:00:00Z", "links": [{"rel": "self", "href": "https://foo-region1.bar.io/keystone/v3/"}], "media-types": [{"base": "application/json", "type": "application/vnd.openstack.identity-v3+json"}]}}
```
---
#### Prep-Node
Run prep-node command on host to be authorized to the management plane (Ensure the cluster blueprint is saved in U/I before doing this):
> [!NOTE]
> Here one would ensure that they provide the actual 1st/2nd region in FQDN (foo-region1.bar.io OR foo-region2.bar.io) and not the Infra Region.
> The below information can also be found if navigated to the `Add a New Host` section in `Infrastructure -> Cluster Hosts` tab in U/I.
```
bash <(curl -s https://ngpc-prod-public-data.s3.us-east-2.amazonaws.com/opencloud/pmo-ctl-setup)
pmo-ctl prep-node
```
```
# bash <(curl -s https://cloud-ctl.s3.us-west-1.amazonaws.com/cloud-ctl-setup)

# cloud-ctl prep-node
Platform9 Account URL: https://foo-region1.bar.io
Username: admin@airctl.localnet
Password: <PASSWORD FROM AIRCTL GET-CREDS COMMAND>
Region [RegionOne]: Region1
Tenant [service]: 
Proxy URL [None]: 
MFA Token [None]: 
...
✓ Platform9 packages installed successfully
```
> [!NOTE]
> Go back to U/I and complete role assignment for the prepped node.
---
#### Installing OpenStack CLI:
```
apt install python3-openstackclient -y
```

###### Authentication: OpenStack CLI:
1. UI: Access & Security -> API Access -> OpenStack RC (Update `OS_REGION_NAME` & `OS_PASSWORD`)
2. `source openstackrc`

#### Upload Image via Glance:
```
openstack image create --insecure --container-format bare --disk-format qcow2 --public --file <path_to_file> <image_name>
```

---
### Stop Management Plane/Regions:
```
airctl stop --config /opt/pf9/airctl/conf/airctl-config.yaml
```
```
# airctl stop --config /opt/pf9/airctl/conf/airctl-config.yaml
 SUCCESS  scaling down management plane foo                                                                                                                                                             
 SUCCESS  scaling down management plane foo-region1 
```

### Start Management Plane/Regions back-up:
```
airctl start --config /opt/pf9/airctl/conf/airctl-config.yaml
```
```
# airctl start --config /opt/pf9/airctl/conf/airctl-config.yaml
 SUCCESS  scaling up management plane foo
 SUCCESS  scaling up management plane foo-region1                                                                                                                                                              
```

### Unconfigure Management Plane/Regions:
```
airctl unconfigure-du --config /opt/pf9/airctl/conf/airctl-config.yaml --force
``` 
Above will remove all configured regions along with infrastructure components like consul, vault, percona, k8sniff, etc.

If re-using the same nodes for management plane to deploy a new build of airctl, ensure to also run below command first before downloading artifacts again
```
rm -rf airctl* install-pmo.sh nodelet* options.json pmo-chart.tgz /opt/pf9/airctl/ .airctl/
```
