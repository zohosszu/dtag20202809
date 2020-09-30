# Module01:

# Preparation of the installation Environment

### Preface:

## Fix firewall settings:

The next steps will be done on bastion.hX.rhaw.io

First we need to open firewall ports on the bastion machine:

```
[root@bastion ~]# firewall-cmd --add-service={dhcp,tftp,http,https,dns} --permanent
```

```
[root@bastion ~]# firewall-cmd --add-port={6443/tcp,22623/tcp,8080/tcp} --permanent
```

```
[root@bastion ~]# firewall-cmd --reload
```

## Setup Bind Named DNS server:

After that we start with configuring the named DNS server:

Comment out the two lines below in /etc/named.conf:

```
[root@bastion ~]# vim /etc/named.conf
```



```
#listen-on port 53 { 127.0.0.1; };
#listen-on-v6 port 53 { ::1; };
```

then we need to allow queries from the VM subnet:

```
allow-query     { localhost;192.168.100.0/24; };
```

after that we need to specify a forwarder for our dns server. this is by default the first ip in our vm network:

```
options { ...
forwarders { 192.168.100.1; };
```

After that we need to define a dns zone inside /etc/named.conf:

```
zone "hX.rhaw.io" IN {
    type master;
    file "hX.rhaw.io.db";
    allow-update { none; };
};
```

After defining this zone we need to create the zone file in: /var/named/hX.rhaw.io.db

```
[root@bastion ~]#  vim /var/named/hX.rhaw.io.db
```

```
$TTL     1D
@        IN  SOA dns.ocp4.hX.rhaw.io. root.hX.rhaw.io. (
                       2019022400 ; serial
                       3h         ; refresh
                       15         ; retry
                       1w         ; expire
                       3h         ; minimum
                                                                             )
                  IN  NS  dns.ocp4.hX.rhaw.io.
dns.ocp4            IN  A   192.168.100.254
bastion            IN CNAME dns.ocp4
bootstrap.ocp4            IN  A   192.168.100.10
master01.ocp4            IN  A   192.168.100.21
master02.ocp4            IN  A   192.168.100.22
master03.ocp4            IN  A   192.168.100.23
etcd-0.ocp4            IN  A   192.168.100.21
etcd-1.ocp4            IN  A   192.168.100.22
etcd-2.ocp4            IN  A   192.168.100.23
api.ocp4               IN  A   192.168.100.254
api-int.ocp4           IN  A   192.168.100.254
*.apps.ocp4            IN  A   192.168.100.254
node01.ocp4            IN  A   192.168.100.31
node02.ocp4            IN  A   192.168.100.32
node03.ocp4            IN  A   192.168.100.33
node04.ocp4            IN  A   192.168.100.34
node05.ocp4            IN  A   192.168.100.35
node06.ocp4            IN  A   192.168.100.36
node07.ocp4            IN  A   192.168.100.37
_etcd-server-ssl._tcp.ocp4    IN  SRV 0 10    2380 etcd-0.ocp4
_etcd-server-ssl._tcp.ocp4      IN      SRV     0 10    2380 etcd-1.ocp4
_etcd-server-ssl._tcp.ocp4      IN      SRV     0 10    2380 etcd-2.ocp4
```

> Please adjust these files to your needs or just take these files exactly as they are!!!

```
[root@bastion ~]# systemctl restart named
```

To test our DNS server we just execute:

```
[root@bastion ~]# dig @localhost -t srv _etcd-server-ssl._tcp.ocp4.hX.rhaw.io
```

Now we need to change the DNS Resolution on bastion Machine and Workstation Machine as well:

On both Machines type in:

```
[root@bastion ~]# nmcli connection show
NAME  UUID                                  TYPE      DEVICE
ens3  191bce9e-d55b-471a-a0fa-c6f060d2e144  ethernet  ens3
```

Now we need to modify the connection to use our new DNS Server on both Virtual Machines:

```
[root@bastion ~]# nmcli connection modify ens3  ipv4.dns "192.168.100.254"
```

After that:

```
[root@bastion ~]# nmcli connection reload
```

```
[root@bastion ~]# nmcli connection up ens3
```

We can test if our Resolution is correct with:

```
[root@bastion ~]# host bootstrap.ocp4.hX.rhaw.io
```

The output should be:

```
bootstrap.ocp4.hX.rhaw.io has address 192.168.100.10
```

When the resolution is not working just reboot your VM and after this it should work.

Now we can step forward.

## Setup DHCP Server:

We need to create / update the /etc/dhcp/dhcpd.conf:

```
[root@bastion ~]# vim /etc/dhcp/dhcpd.conf
```

```
ddns-update-style interim;
 ignore client-updates;
 authoritative;
 allow booting;
 allow bootp;
 allow unknown-clients;
 subnet 192.168.100.0 netmask 255.255.255.0 {
         range 192.168.100.10 192.168.100.100;
         option routers 192.168.100.1;
         option domain-name-servers 192.168.100.254;
         option ntp-servers time.unisza.edu.my;
         option domain-search "hX.rhaw.io","ocp4.hX.rhaw.io";
         filename "pxelinux.0";
         next-server 192.168.100.254;
         host bootstrap { hardware ethernet 52:54:00:e1:78:8a; fixed-address 192.168.100.10; option host-name "bootstrap.ocp4.hX.rhaw.io"; }
         host master01 { hardware ethernet 52:54:00:f1:86:29; fixed-address 192.168.100.21; option host-name "master01.ocp4.hX.rhaw.io"; }
         host master02 { hardware ethernet 52:54:00:af:63:f3; fixed-address 192.168.100.22; option host-name "master02.ocp4.hX.rhaw.io"; }
         host master03 { hardware ethernet 52:54:00:a9:98:dd; fixed-address 192.168.100.23; option host-name "master03.ocp4.hX.rhaw.io"; }
         host node01 { hardware ethernet 52:54:00:9f:95:87; fixed-address 192.168.100.31; option host-name "node01.ocp4.hX.rhaw.io"; }
         host node02 { hardware ethernet 52:54:00:c4:8f:50; fixed-address 192.168.100.32; option host-name "node02.ocp4.hX.rhaw.io"; }
         host node03 { hardware ethernet 52:54:00:fe:e5:e3; fixed-address 192.168.100.33; option host-name "node03.ocp4.hX.rhaw.io"; }
         host node04 { hardware ethernet 52:54:00:f1:79:58; fixed-address 192.168.100.34; option host-name "node04.ocp4.hX.rhaw.io"; }
         host node05 { hardware ethernet 52:54:00:f1:79:59; fixed-address 192.168.100.35; option host-name "node05.ocp4.hX.rhaw.io"; }
         host node06 { hardware ethernet 52:54:00:f1:79:60; fixed-address 192.168.100.36; option host-name "node06.ocp4.hX.rhaw.io"; }
         host node07 { hardware ethernet 52:54:00:f1:79:61; fixed-address 192.168.100.37; option host-name "node07.ocp4.hX.rhaw.io"; }
         

}
```

> Important notice: Please adjust this file as per your environment
> 
> Please ensure that the MAC addresses matches exactly the MAC adresses of the virtual machines we created earlier

## Setup TFTP:

first we need to populate the default file for tftpboot:

```
[root@bastion ~]# mkdir -p  /var/lib/tftpboot/pxelinux.cfg
```

then we need to create the default file with the following content:

```
[root@bastion ~]# vim /var/lib/tftpboot/pxelinux.cfg/default
```

```
default menu.c32
prompt 0
timeout 30
menu title **** OpenShift 4.5 PXE Boot Menu ****

label bootstrap
 kernel /openshift4/4.5.6/rhcos-4.5.6-x86_64-installer-kernel-x86_64
 append ip=dhcp rd.neednet=1 coreos.inst.install_dev=sda console=tty0 console=ttyS0 coreos.inst=yes coreos.inst.image_url=http://192.168.100.254:8080/openshift4/4.5.6/images/rhcos-metal.x86_64.raw.gz coreos.inst.ignition_url=http://192.168.100.254:8080/openshift4/4.5.6/ignitions/bootstrap.ign initrd=/openshift4/4.5.6/rhcos-4.5.6-x86_64-installer-initramfs.img

label master
 kernel /openshift4/4.5.6/rhcos-4.5.6-x86_64-installer-kernel-x86_64
 append ip=dhcp rd.neednet=1 coreos.inst.install_dev=sda console=tty0 console=ttyS0 coreos.inst=yes coreos.inst.image_url=http://192.168.100.254:8080/openshift4/4.5.6/images/rhcos-metal.x86_64.raw.gz coreos.inst.ignition_url=http://192.168.100.254:8080/openshift4/4.5.6/ignitions/master.ign initrd=/openshift4/4.5.6/rhcos-4.5.6-x86_64-installer-initramfs.img

label node
 kernel /openshift4/4.5.6/rhcos-4.5.6-x86_64-installer-kernel-x86_64
 append ip=dhcp rd.neednet=1 coreos.inst.install_dev=sda console=tty0 console=ttyS0 coreos.inst=yes coreos.inst.image_url=http://192.168.100.254:8080/openshift4/4.5.6/images/rhcos-metal.x86_64.raw.gz coreos.inst.ignition_url=http://192.168.100.254:8080/openshift4/4.5.6/ignitions/node.ign initrd=/openshift4/4.5.6/rhcos-4.5.6-x86_64-installer-initramfs.img
```

> Important: Please adjust the IP address to the ip address of your environment

Due to the matter of fact, that we are working in an headless environment, we need to ensure, that the vm's are automatically choose the correct image and ignitonfile for installation. To do so, we need to create 7 files in /var/lib/tftpboot/pxelinux.cfg, with slightly different content:

These files are named by the MAC address for each vm. for example the MAC address of the bootstrap node is:

```
52:54:00:e1:78:8a
```

Then our file needs to be:

```
01-52-54-00-e1-78-8a
```

The content of the file should be:

Bootstrap PXE configuration:

```
default bootstrap
prompt 0
timeout 30
label bootstrap
 kernel /openshift4/4.5.6/rhcos-4.5.6-x86_64-installer-kernel-x86_64
 append ip=dhcp rd.neednet=1 coreos.inst.install_dev=sda console=tty0 console=ttyS0 coreos.inst=yes coreos.inst.image_url=http://192.168.100.254:8080/openshift4/4.5.6/images/rhcos-metal.x86_64.raw.gz coreos.inst.ignition_url=http://192.168.100.254:8080/openshift4/4.5.6/ignitions/bootstrap.ign initrd=/openshift4/4.5.6/rhcos-4.5.6-x86_64-installer-initramfs.img
```

The file for each master  node needs to be:

```
default master
prompt 0
timeout 30
label master
 kernel /openshift4/4.5.6/rhcos-4.5.6-x86_64-installer-kernel-x86_64
 append ip=dhcp rd.neednet=1 coreos.inst.install_dev=sda console=tty0 console=ttyS0 coreos.inst=yes coreos.inst.image_url=http://192.168.100.254:8080/openshift4/4.5.6/images/rhcos-metal.x86_64.raw.gz coreos.inst.ignition_url=http://192.168.100.254:8080/openshift4/4.5.6/ignitions/master.ign initrd=/openshift4/4.5.6/rhcos-4.5.6-x86_64-installer-initramfs.img
```

The file for each node node needs to be:

```
default node
prompt 0
timeout 30
label node
 kernel /openshift4/4.5.6/rhcos-4.5.6-x86_64-installer-kernel-x86_64
 append ip=dhcp rd.neednet=1 coreos.inst.install_dev=sda console=tty0 console=ttyS0 coreos.inst=yes coreos.inst.image_url=http://192.168.100.254:8080/openshift4/4.5.6/images/rhcos-metal.x86_64.raw.gz coreos.inst.ignition_url=http://192.168.100.254:8080/openshift4/4.5.6/ignitions/node.ign initrd=/openshift4/4.5.6/rhcos-4.5.6-x86_64-installer-initramfs.img
```

> Each of the files we now create needs to have a 01- in front and then the MAC Address of each node seperated with a dash!!!

Now we need to copy syslinux for PXE boot:

```
[root@bastion ~]# cp -rvf /usr/share/syslinux/* /var/lib/tftpboot
```

After that start your TFTP server:

```
[root@bastion ~]# systemctl start tftp
```

## Configure Webserver to host Red Hat Core OS images:

First of all we need to change the configuration of the httpd from Listen on port 80 to Listen on Port 8080:

```
[root@bastion ~]# vim /etc/httpd/conf/httpd.conf
```

Search for the Line:

```
Listen 80
```

and turn it into:

```
Listen 8080
```

After that we restart httpd that our changes taking place:

```
[root@bastion ~]# systemctl restart httpd
```

Now we need to create a directory for hosting the kernel and initramfs for PXE boot:

```
[root@bastion ~]# mkdir -p /var/lib/tftpboot/openshift4/4.5.6/
```

access this directory:

```
[root@bastion ~]# cd /var/lib/tftpboot/openshift4/4.5.6/
```

and download the kernel file to this directory:

```
[root@bastion ~]# wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.5/4.5.6/rhcos-4.5.6-x86_64-installer-kernel-x86_64
```

Then the CoreOS Installer initramfs image:

```
[root@bastion ~]# wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.5/4.5.6/rhcos-4.5.6-x86_64-installer-initramfs.x86_64.img
```

Now we ned to relabel the files for selinux:

```
[root@bastion ~]# restorecon -RFv .
```

Next we need to host the Red Hat Core OS metal BIOS image:

```
[root@bastion ~]# mkdir -p /var/www/html/openshift4/4.5.6/images/
```

```
[root@bastion ~]# cd  /var/www/html/openshift4/4.5.6/images/
```

```
[root@bastion ~]# wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.5/4.5.6/rhcos-metal.x86_64.raw.gz
```

```
[root@bastion ~]# restorecon -RFv .
```

## Setup HAProxy as Loadbalancer:

We are going step by step to the end of our preparations. The last service we need to configure is the haproxy service.

Use the following code snippet and place it in /etc/haproxy. Please make a backup of your default haproxy.conf before.

```
[root@bastion ~]# cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.default
```

```
[root@bastion ~]# vim /etc/haproxy/haproxy.cfg
```

/etc/haproxy/haproxy.cfg:

```
defaults
    timeout connect         5s
    timeout client          30s
    timeout server          30s
    log                     global

frontend kubernetes_api
    bind 0.0.0.0:6443
    default_backend kubernetes_api

backend kubernetes_api
    balance roundrobin
    option ssl-hello-chk
    server bootstap bootstrap.ocp4.hX.rhaw.io:6443 check
    server master01 master01.ocp4.hX.rhaw.io:6443 check
    server master02 master02.ocp4.hX.rhaw.io:6443 check
    server master03 master03.ocp4.hX.rhaw.io:6443 check

frontend machine_config
    bind 0.0.0.0:22623
    default_backend machine_config

backend machine_config
    balance roundrobin
    option ssl-hello-chk
    server bootstrap bootstrap.ocp4.hX.rhaw.io:22623 check
    server master01 master01.ocp4.hX.rhaw.io:22623 check
    server master02 master02.ocp4.hX.rhaw.io:22623 check
    server master03 master03.ocp4.hX.rhaw.io:22623 check

frontend router_https
    bind 0.0.0.0:443
    default_backend router_https

backend router_https
    balance roundrobin
    option ssl-hello-chk
    server node01 node01.ocp4.hX.rhaw.io:443 check
    server node02 node02.ocp4.hX.rhaw.io:443 check
    server node03 node03.ocp4.hX.rhaw.io:443 check
    server node04 node04.ocp5.hX.rhaw.io:443 check
    server node05 node05.ocp6.hX.rhaw.io:443 check
    server node06 node06.ocp7.hX.rhaw.io:443 check

frontend router_http
    mode http
    option httplog
    bind 0.0.0.0:80
    default_backend router_http

backend router_http
    mode http
    balance roundrobin
    server node01 node01.ocp4.hX.rhaw.io:80 check
    server node02 node02.ocp4.hX.rhaw.io:80 check
    server node03 node03.ocp4.hX.rhaw.io:80 check
    server node04 node04.ocp4.hX.rhaw.io:80 check
    server node04 node04.ocp4.hX.rhaw.io:443 check
    server node05 node05.ocp4.hX.rhaw.io:443 check
    server node06 node06.ocp4.hX.rhaw.io:443 check
```

> Important: Please adjust this file according to your environment if needed.

Now we need to configure SElinux to use custom ports in SELinux:

```
[root@bastion ~]# semanage port  -a 22623 -t http_port_t -p tcp
```

```
[root@bastion ~]# semanage port -a 6443 -t http_port_t -p tcp
```

```
[root@bastion ~]# semanage port -a 32700 -t http_port_t -p tcp
```

## Install Local Registry

For our disconnected Installation we need to install an own local Registry on our bastion Machine. For that we first need to install the openshift client tools:

```
[root@bastion ~]# wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.5.6/openshift-client-linux-4.5.6.tar.gz
```

After downloading the client tools we need to extract them:

```
[root@bastion ~]# tar -xvf openshift-client-linux-4.5.6.tar.gz
```

Then we need to copy the files to the proper location on our bastion machine:

```
[root@bastion ~]# cp -v oc kubectl /usr/local/bin/
```

If not already done we need to install several packages:

```
[root@bastion ~]# yum -y install podman httpd-tools
```

Now we need to create the needed folders for our registry

```
[root@bastion ~]# mkdir -p /opt/registry/{auth,certs,data}
```

Now we need to Provide a certificate for the registry. If we do not have an existing, trusted certificate authority, we can generate a self-signed certificate:

```
[root@bastion ~]# cd /opt/registry/certs
```

```
[root@bastion ~]# openssl req -newkey rsa:4096 -nodes -sha256 -keyout domain.key -x509 -days 365 -out domain.crt
```

The procedure will ask several questions that need to be answered:

| Country Name (2 letter code)                          | Specify the two-letter ISO country code for your location. See the [ISO 3166 country codes](https://www.iso.org/iso-3166-country-codes.html) standard.       |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| State or Province Name (full name)                    | Enter the full name of your state or province.                                                                                                               |
| Locality Name (eg, city)                              | Enter the name of your city.                                                                                                                                 |
| Organization Name (eg, company)                       | Enter your company name.                                                                                                                                     |
| Organizational Unit Name (eg, section)                | Enter your department name.                                                                                                                                  |
| Common Name (eg, your name or your server’s hostname) | Enter the host name for the registry host. Ensure that your hostname is in DNS and that it resolves to the expected IP address.                              |
| Email Address                                         | Enter your email address. For more information, see the [req](https://www.openssl.org/docs/man1.1.1/man1/req.html) description in the OpenSSL documentation. |

After creating the registry we need to create a username and password for our registry

```
[root@bastion ~]# htpasswd -bBc /opt/registry/auth/htpasswd <user_name> <password>
```

Username and Password should be: student and redhat

The next step is to run our local registry with the following command:

```
[root@bastion ~]# podman run --name mirror-registry -p 5000:5000 \ 
     -v /opt/registry/data:/var/lib/registry:z \
     -v /opt/registry/auth:/auth:z \
     -e "REGISTRY_AUTH=htpasswd" \
     -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
     -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
     -v /opt/registry/certs:/certs:z \
     -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
     -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
     -d docker.io/library/registry:2
```

After done this we need to open several ports on our registry node:

```
[root@bastion ~]# firewall-cmd --add-port=5000/tcp --zone=internal --permanent 
```

```
[root@bastion ~]# # firewall-cmd --add-port=5000/tcp --zone=public   --permanent
```

```
[root@bastion ~]# firewall-cmd --reload
```

Now we need to add the self created certificate to the the local trust:

```
[root@bastion ~]# cp /opt/registry/certs/domain.crt /etc/pki/ca-trust/source/anchors/
```

```
[root@bastion ~]# update-ca-trust
```

Now we have created all of our bastion. the next step is to prepare the installation from the Openshift perspective

## Configure OpenShift installer and CLI binary:

From now on, unless otherwise stated, all steps will be performed on bastion.hX.rhaw.io

We need to login with ssh and the username and password provided through the instructor:

```
ssh root@bastion.hX.rhaw.io
```

```

Now we need to create a SSH key pair to access to use later to access the CoreOS nodes

```
[root@bastion ~]# ssh-keygen -t rsa -N "" -f /root/.ssh/id_rsa
```

## Set up the ignition files

We have to create the ignition files they will be used for the installation:

First we start with the install-config-base.yaml file

```
[root@bastion ~]# vim install-config-base.yaml
```

The output is something like this:

```
BGVtbYk3ZHAtqXs=
```

Now we need to make a copy in json format of the original pull secret file downloaded from cloud.redhat.com:

```
[root@bastion ~]# cat ./pull-secret.text | jq .  > /root/ocp4/pull-secret-local.text
```

The file looks similar to the example below:

```
{
  "auths": {
    "cloud.openshift.com": {
      "auth": "b3BlbnNo...",
      "email": "you@hX.rhaw.io"
    },
    "quay.io": {
      "auth": "b3BlbnNo...",
      "email": "you@hX.rhaw.io"
    },
    "registry.connect.redhat.com": {
      "auth": "NTE3Njg5Nj...",
      "email": "you@hX.rhaw.io"
    },
    "registry.redhat.io": {
      "auth": "NTE3Njg5Nj...",
      "email": "you@hX.rhaw.io"
    }
  }
}
```

Now we need to edit the new file like this:

```
[root@bastion ~]# vim /root/ocp4/pull-secret-local.text
```

We just need to add a section to this file that describes the local registry:

```
"auths": {
...
 "<bastion.hX.rhaw.io:5000": {
 "auth": "BGVtbYk3ZHAtqXs=",
 "email": "root@hX.rhaw.io"
 },
...
```

After we have done this we need to mirror the needed images to our registry.

We need to set some variables so that we can mirror the correct content:

```
[root@bastion ~]# export OCP_RELEASE=4.5.6
```

```
[root@bastion ~]# export LOCAL_REGISTRY='bastion.hX.rhaw.io:5000'
```

```
[root@bastion ~]# export LOCAL_REPOSITORY='ocp4/openshift4' 
```

```
[root@bastion ~]# export PRODUCT_REPO='openshift-release-dev'
```

```
[root@bastion ~]# export LOCAL_SECRET_JSON='/root/ocp4/pull-secret-local.text'
```

```
[root@bastion ~]# export RELEASE_NAME="ocp-release"
```

After we have done this we can now mirror the images ... +/- 99 images.

```
[root@bastion ~]# oc adm -a ${LOCAL_SECRET_JSON} release mirror \
     --from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE} \
     --to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} \
     --to-release image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}
```

After the mirroring has been done we can create an installation program that is based on the content that we have just mirrored:

```
[root@bastion ~]# oc adm -a ${LOCAL_SECRET_JSON} release extract --command=openshift-install "${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}"
```

the last step is to optain the ssh public we earlier created:

```
[root@bastion ~]# cat /root/.ssh/id_rsa.pub
```

Next we need to create the ignition files that will be used during the installation:

```
[root@bastion ~]# vim /root/ocp4/install-config-base.yaml
```

Now we need to create the install-config-base.yaml file:

irst we start with the install-config-base.yaml file

```
[root@bastion ~]# vim install-config-base.yaml
```

```
apiVersion: v1
baseDomain: hX.rhaw.io
compute:
- hyperthreading: Enabled
  name: node
  replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: ocp4
networking:
  clusterNetworks:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
pullSecret: 'GET FROM cloud.redhat.com'
sshKey: 'SSH PUBLIC KEY'
imageContentSources:
 - mirrors:
 - bastion.hX.rhaw.io:5000/<repo_name>/release
 source: quay.io/openshift-release-dev/ocp-release
- mirrors:
```

Please adjust this file to your needs.

> The pull secret can be obtained after accessing: https://cloud.redhat.com
> 
> Please login with your RHNID and your password.
> 
> The pull secret can be found when access the following link:
> 
> https://cloud.redhat.com/openshift/install/metal/user-provisioned

To obtain this key please execute:

```
[root@bastion ~]# cat /root/.ssh/id_rsa.pub
```

Copy the content of the output into the sshKey parameter: Don't miss the quotes at the beginning and the end of the cpoied string!

Create the direcory ocp4

```
[root@bastion ~]# mkdir -p ocp4
```

And change into it

```
[root@bastion ocp4]# cd ocp4
```

Copy the install-config-base.yaml file into the ocp4 directory and rename it to install-config.yaml

```
[root@bastion ocp4]# cp ../install-config-base.yaml install-config.yaml
```

Don't forget to copy this file this is very important!!! If this file is missing, the creation of the ignition files will fail!

> Everytime you recreate the ignition files you need to ensure that the ocp4 directory is empty except the install-config-base.yaml file and the manifest files have to be recreated and modifyied. Very Important the .openshift_install_state.json file needs to be deleted before you recreate the ignition file. This file contains the installation certificates and can damage your installation when you use old certificates in new ignition files.

Because we want to prevent pods from being scheduled on the control plane machines (masters), we have to modifiy the `manifests/cluster-scheduler-02-config.yml` Kubernetes manifest file.

To create the Kubernetes manifest files run:

```
[root@bastion ocp4]# openshift-install create manifests
INFO Consuming Install Config from target directory 
WARNING Making control-plane schedulable by setting MastersSchedulable to true for Scheduler cluster settings
```

Directory content looks now like this (the install-config.yaml is gone!):

```
[root@bastion ocp4]# tree /root/ocp4
/root/ocp4
├── manifests
│   ├── 04-openshift-machine-config-operator.yaml
│   ├── cluster-config.yaml
│   ├── cluster-dns-02-config.yml
│   ├── cluster-infrastructure-02-config.yml
│   ├── cluster-ingress-02-config.yml
│   ├── cluster-network-01-crd.yml
│   ├── cluster-network-02-config.yml
│   ├── cluster-proxy-01-config.yaml
│   ├── cluster-scheduler-02-config.yml
│   ├── cvo-overrides.yaml
│   ├── etcd-ca-bundle-configmap.yaml
│   ├── etcd-client-secret.yaml
│   ├── etcd-host-service-endpoints.yaml
│   ├── etcd-host-service.yaml
│   ├── etcd-metric-client-secret.yaml
│   ├── etcd-metric-serving-ca-configmap.yaml
│   ├── etcd-metric-signer-secret.yaml
│   ├── etcd-namespace.yaml
│   ├── etcd-service.yaml
│   ├── etcd-serving-ca-configmap.yaml
│   ├── etcd-signer-secret.yaml
│   ├── kube-cloud-config.yaml
│   ├── kube-system-configmap-root-ca.yaml
│   ├── machine-config-server-tls-secret.yaml
│   └── openshift-config-secret-pull-secret.yaml
└── openshift
    ├── 99_kubeadmin-password-secret.yaml
    ├── 99_openshift-cluster-api_master-user-data-secret.yaml
    ├── 99_openshift-cluster-api_node-user-data-secret.yaml
    ├── 99_openshift-machineconfig_99-master-ssh.yaml
    └── 99_openshift-machineconfig_99-node-ssh.yaml

2 directories, 30 files
```

We have to set the value of the parameter `mastersSchedulable` from true to false

```
[root@bastion ocp4]# sed -i 's/true/false/' manifests/cluster-scheduler-02-config.yml
```

Now we will create the ignition files:

```
[root@bastion ocp4]# openshift-install create ignition-configs
INFO Consuming Install Config from target directory 
INFO Consuming node Machines from target directory 
INFO Consuming Master Machines from target directory 
INFO Consuming Openshift Manifests from target directory 
INFO Consuming Common Manifests from target directory 
```

The content of the directory ocp4 has now this content (the directoies openshift and manifests are gone!)

```
[root@bastion ocp4]# tree /root/ocp4
/root/ocp4
├── auth
│   ├── kubeadmin-password
│   └── kubeconfig
├── bootstrap.ign
├── master.ign
├── metadata.json
└── node.ign
```
```

Don't forget to copy this file this is very important!!! If this file is missing, then the creation of the ignition files will fail!!!

> Everytime you recreate the ignition files you need to ensure that the ocp4 directory is empty except the install-config-base.yaml file. Very Important the .openshift_install_state.json file needs to be deleted before you recreate the ignition file. This file contains the installation certificates and can damage your installation when you use old certificates in new ignition files.

```
[root@bastion ~]# openshift-install create ignition-configs
```

```
drwxr-xr-x. 3 root root     195 29. Nov 18:01 .
dr-xr-x---. 9 root root    4096 29. Nov 18:00 ..
drwxr-xr-x. 2 root root      50 29. Nov 18:01 auth
-rw-r--r--. 1 root root  288789 29. Nov 18:01 bootstrap.ign
-rw-r--r--. 1 root root    3716 24. Nov 23:58 install-config-base.yaml
-rw-r--r--. 1 root root    1825 29. Nov 18:01 master.ign
-rw-r--r--. 1 root root      96 29. Nov 18:01 metadata.json
-rw-r--r--. 1 root root   58088 29. Nov 18:01 .openshift_install.log
-rw-r--r--. 1 root root 1190917 29. Nov 18:01 .openshift_install_state.json
-rw-r--r--. 1 root root    1825 29. Nov 18:01 node.ign
```

Now we need to copy the files to our httpd server:

```
[root@bastion ~]# mkdir -p /var/www/html/openshift4/4.2.0/ignitions
```

```
[root@bastion ~]# cp -v *.ign /var/www/html/openshift4/4.2.0/ignitions/
```

```
[root@bastion ~]# restorecon -RFv /var/www/html/
```

Now we are done with the installation and can start the initial cluster installation.

```
[root@bastion ocp4]# systemctl enable --now haproxy.service dhcpd httpd tftp named
```

> Important: ensure every time that haproxy is up and running. Sometimes during reboot of your service machine it is not coming up.

To ensure type:

```
[root@bastion ocp4]# systemctl status haproxy
```

If the state is failed then type:

```
[root@bastion ocp4]# systemctl restart haproxy
```

Re-check again:

```
[root@bastion ocp4]# systemctl status haproxy
```

Now we are able to install our virtual machines for installing openshift cluster. The Procedure of the installation is similar to the connected installation.
