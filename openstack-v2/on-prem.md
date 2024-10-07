# Platform9 On-Prem Installation

## Prep / Configuration
### On all management plane nodes
#### Install  cgroup-tools
```
apt-get update -y && apt-get install cgroup-tools -y
```

#### Update OpenSSL Version to 3.0.7 for Ubuntu 22.04
```
curl --user-agent "d1307db89fe74daa83ebd17a71218198" https://pf9-airctl.s3-accelerate.amazonaws.com/openssl-smcp-ubuntu/openssl_3.0.7-1_amd64.deb --output openssl_3.0.7-1_amd64.deb
sudo dpkg -i openssl_3.0.7-1_amd64.deb
cd /etc/ld.so.conf.d/
echo "/usr/local/ssl/lib64" > openssl-3.0.7.conf
ldconfig -v
sed -i 's/\(^PATH="[^"]*\)"/\1:\/usr\/local\/ssl\/bin"/' /etc/environment
source /etc/environment
ln -sf /usr/local/ssl/bin/openssl /usr/bin/openssl
cd
```

##### Verify OpenSSL Version
```
openssl version
```

### On a single management plane node
#### Download Platform9 Artifacts & Execute Installer
```
curl --user-agent "d1307db89fe74daa83ebd17a71218198" https://pf9-airctl.s3-accelerate.amazonaws.com/v-5.12.0-3407038/index.txt | grep -e airctl -e install-pmo.sh -e nodelet-deb.tar.gz -e nodelet.tar.gz -e pmo-chart.tgz -e options.json | awk '{print "curl -sS --user-agent \"d1307db89fe74daa83ebd17a71218198\" \"https://pf9-airctl.s3-accelerate.amazonaws.com/v-5.12.0-3407038/" $NF "\" -o /root/" $NF}' | bash
```

```
chmod +x ./install-pmo.sh
```


```
./install-pmo.sh v-5.12.0-3407038
Extracting airctl tar.gz
Extracting airctl scripts, conf
Copying nodelet rpm and deb tar.gz
Copying options file
Done
```

```
echo 'export PATH=$PATH:/opt/pf9/airctl' >> ~/.bashrc
source ~/.bashrc
```



#### Airctl Configuration
> [!NOTE]
> The following airctl configure command is configured as follows: (172.29.21.157 172.29.20.223 172.29.20.157 3 master nodes) (172.29.20.175 VIP for management cluster) (172.29.21.1 VIP for External IP):

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
> [!NOTE] 
> Ensure to select install driver as openstack. The FQDN you will provide in the above command prompt will be for the Infra region which gets populated into duFqdn field in airctl-config.yaml.



#### Post Command Completion
> Edit `/opt/pf9/airctl/conf/airctl-config.yaml` and populate the field `secduFqdn` with the actual region URLs and edit duRegion fields in a space separated manner.
> Note that the first region name in duRegion field should be Infra. The other regions can be named as preferred. Here we have set is as Region1
```
$ cat /opt/pf9/airctl/conf/airctl-config.yaml | grep -ie duFqdn -ie secduFqdn -ie duRegion
duFqdn: foo.bar.io
secduFqdn: foo-region1.bar.io
duRegion: Infra Region1
```
> [!IMPORTANT]
> In above, foo.bar.io corresponds to Infra region which will only have Keystone service running. The Nova, Neutron, Glance and rest of the services would run on the main regions. From the example, foo-region1.bar.io corresponding to Region1


Additionally, add below parameters in `/opt/pf9/airctl/conf/nodelet-bootstrap-config.yaml`. Change settings as required if interface is different.
```
masterVipEnabled: true
masterVipInterface: ens3
masterVipVrouterId: 119
```


## Deployment

#### Create Nodelet Multi-Master K8s cluster
```
airctl advanced-ddu create-mgmt --config /opt/pf9/airctl/conf/airctl-config.yaml
```

Example output of cluster post-creation:
```
# kubectl get nodes --kubeconfig='/etc/pf9/kube.d/kubeconfigs/admin.yaml'
NAME            STATUS   ROLES    AGE     VERSION
172.29.20.157   Ready    master   4m29s   v1.29.2
172.29.20.223   Ready    master   5m41s   v1.29.2
172.29.21.157   Ready    master   5m42s   v1.29.2
# kubectl get pods -A --kubeconfig='/etc/pf9/kube.d/kubeconfigs/admin.yaml'
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-6f89d5c87f-44r7x   1/1     Running   0          5m54s
kube-system   calico-node-8zs4q                          1/1     Running   0          5m52s
kube-system   calico-node-l8qnp                          1/1     Running   0          5m53s
kube-system   calico-node-ztjzf                          1/1     Running   0          4m40s
kube-system   calico-typha-79f65c5d4c-2s9fg              1/1     Running   0          13s
kube-system   calico-typha-79f65c5d4c-582bd              1/1     Running   0          5m13s
kube-system   calico-typha-79f65c5d4c-mjzgb              1/1     Running   0          5m54s
kube-system   calico-typha-autoscaler-6cc98c9474-4ps8n   1/1     Running   0          5m54s
kube-system   coredns-58b6bd776d-2gnr6                   1/1     Running   0          4m29s
kube-system   k8s-master-172.29.20.157                   3/3     Running   0          3m16s
kube-system   k8s-master-172.29.20.223                   3/3     Running   0          4m29s
kube-system   k8s-master-172.29.21.157                   3/3     Running   0          5m1s
kube-system   kube-dns-autoscaler-5d49f47844-7whbj       1/1     Running   0          5m32s
```
> [!NOTE]
> Please refer to `/var/log/pf9/nodelet.log` for cluster creation troubleshooting

#### Run airctl init
```
airctl init --config /opt/pf9/airctl/conf/airctl-config.yaml
```

### Run airctl start
```
airctl start --config /opt/pf9/airctl/conf/airctl-config.yaml
```

Example:
```
# airctl start --config /opt/pf9/airctl/conf/airctl-config.yaml
 INFO  openstack management plane creation started
 SUCCESS  generating certs and config... 
 SUCCESS  setting up base infrastructure...
 INFO  kplane setup done, creating management plane 
 SUCCESS  starting multi-region openstack deployment... 
 INFO  openstack management plane created - the services will take a while to start
```

> [!NOTE]
> It will take up to 45 minutes for all the services to get deployed for Infra and 1 region. 



## Monitoring Installation Progress
We can monitor the progress of the DU deployment by looking at the logs of the du-install pod e.g.
```
export KUBECONFIG=/etc/pf9/kube.d/kubeconfigs/admin.yaml
kubectl get pod -A | grep du-install
kubectl logs -n <ns> <pod name> -f
```
Example:
```
# kubectl get pods -n foo-kplane | grep du-install
du-install-foo-bmqqd                        0/1     Completed   0          108m
du-install-foo-region1-f7fdw                0/1     Completed   0          98m
```
> [!NOTE]
> Please refer to airctl-logs/airctl.log for logs in case of any issues with airctl start command

Also, users can check the status using airctl status command. Sample output
```
# airctl status --config /opt/pf9/airctl/conf/airctl-config.yaml
------------- deployment details ---------------
fqdn:                foo.bar.io
cluster:             foo-kplane.bar.io
task state:          ready
-------- region service status ----------
task state:        ready
desired services:  22
ready services:    22
```

## Obtaining UI Credentials
```
airctl get-creds --config /opt/pf9/airctl/conf/airctl-config.yaml
```

Example:
```# airctl get-creds --config /opt/pf9/airctl/conf/airctl-config.yaml 
email:     admin@airctl.localnet
password:  qaChMBjgkghucTnv
```


## Preparing Hosts for Authorization to Management Plane

#### Updating /etc/hosts/
> [!NOTE]
> On the local machine **AND** any new host that wants to be authorized to the created management plane, update the /etc/hosts file with the IP address of the management plane VM (172.29.21.1 in this case) and the FQDN corresponding to it.
```
$ cat /etc/hosts | grep foo
<VIP for externalIP>         foo-region1.bar.io
```

#### Add Management Plane Certificate
> [!NOTE]
> Since the DU uses self-signed certificates, the API calls from `pmo-ctl` will fail. You need to add the DU certificate using the following commands for Ubuntu 22 on all hypervisor hosts:

```
# Get the DUs self-signed certificate
openssl s_client -showcerts -connect $DU_URL:443 2>/dev/null </dev/null | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > du.crt

# Copy it to trusted store
cp du.crt /usr/local/share/ca-certificates

# Refresh CA certs
sudo update-ca-certificates

# Ensure curl does not complain about the certificate
curl https://$DU_URL
```

Example:
```
root@sanchit-pmov2-host:~# export DU_URL=foo-region1.bar.io 
root@sanchit-pmov2-host:~# openssl s_client -showcerts -connect $DU_URL:443 2>/dev/null </dev/null | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > du.crt
root@sanchit-pmov2-host:~# cp du.crt /usr/local/share/ca-certificates
root@sanchit-pmov2-host:~# sudo update-ca-certificates
Updating certificates in /etc/ssl/certs...
rehash: warning: skipping ca-certificates.crt,it does not contain exactly one certificate or CRL
1 added, 0 removed; done.
Running hooks in /etc/ca-certificates/update.d...
done.
root@sanchit-pmov2-host:~# curl https://$DU_URL
<html>
<head><title>302 Found</title></head>
<body>
<center><h1>302 Found</h1></center>
<hr><center>nginx/1.26.2</center>
</body>
</html>
root@sanchit-pmov2-host:~# curl https://$DU_URL/keystone/v3
{"version": {"id": "v3.14", "status": "stable", "updated": "2020-04-07T00:00:00Z", "links": [{"rel": "self", "href": "https://foo-region1.bar.io/keystone/v3/"}], "media-types": [{"base": "application/json", "type": "application/vnd.openstack.identity-v3+json"}]}}
```



## Unconfigure Management Plane
As of now, airctl unconfigure-du will only delete the infra deployments like kplane, consul, vault, k8sniff etc and the Infra region i.e. foo.
```
airctl unconfigure-du --force --config /opt/pf9/airctl/conf/airctl-config.yaml
```

Delete the other region namespaces manually:
```# kubectl delete ns foo-region1
namespace "foo-region1" deleted
```
After this, all the region namespaces get stuck in terminating state. 
```
# kubectl get ns | grep foo
foo                     Terminating   123m
foo-region1             Terminating   120m
```
You have to go and delete the perconaxtradbclusters.pxc.percona.com object and then edit them and remove the finalizers for each region.
```
# kubectl get perconaxtradbclusters.pxc.percona.com -A
NAMESPACE     NAME                ENDPOINT                                STATUS   PXC   PROXYSQL   HAPROXY   AGE
foo-region1   percona-db-pxc-db   percona-db-pxc-db-haproxy.foo-region1   ready    1                1         42m
foo           percona-db-pxc-db   percona-db-pxc-db-haproxy.foo           error    1                1         42m
# kubectl delete perconaxtradbclusters percona-db-pxc-db -n foo &
# kubectl delete perconaxtradbclusters percona-db-pxc-db -n foo-region1 &
# kubectl edit perconaxtradbclusters.pxc.percona.com percona-db-pxc-db -n foo
# kubectl edit perconaxtradbclusters.pxc.percona.com percona-db-pxc-db -n foo-region1
Remove all 3 under `finalizers`:
  finalizers:
  - delete-pxc-pods-in-order
  - delete-proxysql-pvc
  - delete-pxc-pvc
# kubectl get perconaxtradbclusters.pxc.percona.com -A
No resources found
# kubectl get ns | grep foo
~#
```
