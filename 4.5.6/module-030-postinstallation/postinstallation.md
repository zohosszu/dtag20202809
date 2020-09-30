After we have done the installation we can check the cluster is up and running type in the following command:

```sh
[root@bastion ~]# export KUBECONFIG=/root/ocp4/auth/kubeconfig
```

```sh
[root@bastion ~]# oc whoami
system:admin
```

```sh
[root@bastion ~]# oc get nodes
NAME       STATUS   ROLES           AGE   VERSION
master01   Ready    master,node   37h   v1.16.2
master02   Ready    master,node   37h   v1.16.2
master03   Ready    master,node   37h   v1.16.2
node01   Ready    node          37h   v1.16.2
node02   Ready    node          37h   v1.16.2
node03   Ready    node          21h   v1.16.2
```

You should get an output of six machines in state Ready.

## Troubleshooting: Pending  Certificates

When you add machines to a cluster, two pending certificates signing request (CSRs) are generated for each machine that you added. You must verify that these CSRs are approved or, if necessary, approve them yourself.

```sh
[root@bastion ~]# oc get nodes
NAME       STATUS      ROLES           AGE   VERSION
master01   Ready       master,node   37h   v1.16.2
master02   Ready       master,node   37h   v1.16.2
master03   Ready       master,node   37h   v1.16.2
node01   Ready       node          37h   v1.16.2
node02   NotReady    node          37h   v1.16.2
node03   NotReady    node          21h   v1.16.2
...
```

The output lists all of the machines that we created.

Now we need to review the pending certificate signing requests (CSRs) and ensure that the you see a client and server request with `Pending` or `Approved` status for each machine that you added to the cluster:

```sh
[root@bastion ~]# oc get csr
NAME        AGE     REQUESTOR                                                                   CONDITION
csr-8b2br   15m     system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
csr-8vnps   15m     system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
csr-bfd72   5m26s   system:node:ip-10-0-50-126.us-east-2.compute.internal                       Pending
csr-c57lv   5m26s   system:node:ip-10-0-95-157.us-east-2.compute.internal                       Pending
```

> |
> Because the CSRs rotate automatically, approve your CSRs within an hour of adding the machines to the cluster. If you do not approve them within an hour, the certificates will rotate, and more than two certificates will be present for each node. You must approve all of these certificates. After you approve the initial CSRs, the subsequent node client CSRs are automatically approved by the cluster kube-controller-manager. You must implement a method of automatically approving the kubelet serving certificate requests.

Now we need to approve pending certificates:

```sh
[root@bastion ~]# oc adm certificate approve csr-bfd72
```

Tip:
To approve all pending certificates run the folloing command:

```sh
[root@bastion ~]# oc get csr -o name | xargs oc adm certificate approve
```

After that we can check the csr status again and validate that they are all "Approved,Issued":

```sh
[root@bastion ~]# oc get csr
```

## Completing installation on User Provisioned Infrastructure:

After we complete the operator configuration, you can finish installing the cluster on infrastructure that you provide.

We need to confirm that all components are up and running.

```sh
[root@bastion ~]# watch -n5 oc get clusteroperators
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
authentication                             4.5.6     True        False         False      10m
cloud-credential                           4.5.6     True        False         False      22m
cluster-autoscaler                         4.5.6     True        False         False      21m
console                                    4.5.6     True        False         False      10m
dns                                        4.5.6     True        False         False      21m
image-registry                             4.5.6     True        False         False      16m
ingress                                    4.5.6     True        False         False      16m
insights                                   4.5.6     True        False         False      19m
kube-apiserver                             4.5.6     True        False         False      18m
kube-controller-manager                    4.5.6     True        False         False      22m
kube-scheduler                             4.5.6     True        False         False      22m
machine-api                                4.5.6     True        False         False      18m
machine-config                             4.5.6     True        False         False      18m
marketplace                                4.5.6     True        False         False      18m
monitoring                                 4.5.6     True        False         False      16m
network                                    4.5.6     True        False         False      21m
node-tuning                                4.5.6     True        False         False      21m
openshift-apiserver                        4.5.6     True        False         False      17m
openshift-controller-manager               4.5.6     True        False         False      14m
openshift-samples                          4.5.6     True        False         False      21m
operator-lifecycle-manager                 4.5.6     True        False         False      21m
operator-lifecycle-manager-catalog         4.5.6     True        False         False      21m
operator-lifecycle-manager-packageserver   4.5.6     True        False         False      21m
service-ca                                 4.5.6     True        False         False      16m
service-catalog-apiserver                  4.5.6     True        False         False      16m
service-catalog-controller-manager         4.5.6     True        False         False      16m
storage                                    4.5.6     True        False         False      17m
```

  When all of the cluster Operators are available (the kube-apiserver operator is last in state PROGRESSING=True and takes roughly 15min to finish), we can complete the installation.

> The Ignition config files that the installation program generates contain certificates that expire after 24 hours. You must keep the cluster running for 24 hours in a non-degraded state to ensure that the first certificate rotation has finished.
