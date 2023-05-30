# Auto Certificates Renewal

## Context

### Problem

When k8s nodes are created by eksa, they use certificate pairs to communicate with each other. But the certificates they use will expire in 1 year, and after 1 year, the k8s nodes will not be able to communicate with each other and the cluster will break down.

To address this issue, the following CX needs to be implemented.


>As a user who deployed K8s cluster with EKS-A, I want an option to automatically renew the certificates in etcd and control plane nodes before expiry in 1 year.

### External ETCD node certificates

* etcd node’s certificates are stored in /etc/etcd/pki (/var/lib/etcd/pki in bottlerocket)
* each certificate comes in pair: key and crt
* all etcd nodes contain the same key pair names no matter if it’s the first node (not in term of contents)

list of all the certificate files:

1. apiserver-etcd-client pair: generated by etcdadm,  apiserver can use it to access etcd, **but it’s not used**
2. ca pair: the only pair which is not generated by etcdadm, it comes from cloud-init script.
3. etcdctl-etcd-client pair: generated by etcdadm during `etcdadm join <endpoint>`
4. peer pair: generated by etcdadm `etcdadm join <endpoint>`
5. server pair: generated by etcdadm `etcdadm join <endpoint>`

### CP node certificates

* all certificates are located in /etc/kubernetes/pki  (/var/lib/kubeadm/pki in bottlerocket)
* all cp nodes contains the same key pair names

list of all the certificate files:

1. apiserver-etcd-client pair: from cloud-init script, but the source is etcdadm controller 
2. ca pair: from cloud-init script
3. front-proxy-ca pair: from cloud-init script
4. sa pair: from cloud-init script
5. etcd/ca.crt: from cloud-init script
6. apiserver-kubelet-client pair: from kubeadm
7. apiserver pair: from kubeadm
8. front-proxy-client pair: from kubeadm

The above certificate files are described in the official doc: https://kubernetes.io/docs/setup/best-practices/certificates/#all-certificates

### Worker node certificates

1. just ca.crt in /etc/kubernetes/pki, which should be generated after `kubeadm join`

## Goal

1. Auto renew all the above mentioned certificates except etcd CA and kube CA, which last for 10 years.

### Observation

1. We can renew all etcd’s certs by simply **recreating the external etcd machine**. If doing manually, we just need to delete the auto generated certs and run `sudo etcdadm join phase certificates`
2. We can renew all cp’s certs besides apiserver-etcd-client by **recreating the cp machine.** As we can see from the following diagram that the apiserver-etcd-client is generated by Etcdadm Controller, so this cert cannot be renewed by just recreating the cp machine.
3. No need to do auto renew certs in worker nodes.

## Solution

Etcd can be deployed in [two topologies.](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/) Certs renew mechanism is slightly different in those two scenarios.

### Stacked etcd Nodes

In Stacked etcd nodes topology, etcd is running inside the cp node. 

We can renew all certificates by just redeploying cp nodes

1. Expose [rolloutBefore.certificatesExpiryDays](https://cluster-api.sigs.k8s.io/tasks/certs/auto-rotate-certificates-in-kcp.html) field from KubeadmControlPlane in EKSA cluster spec. Add pass the field to KubeadmControlPlane

### External etcd Nodes

In external etcd nodes topology, etcd is running in a separate machine besides cp machine. In addition to roll out cp nodes, we also need to roll out etcd nodes, with the following changes on top of above:

1. Add rolloutBefore.certificatesExpiryDays field to [EtcdadmCluster](https://github.com/aws/etcdadm-controller/blob/main/api/v1beta1/etcdadmcluster_types.go#L38), exposing and passing it from EKSA. The business logic of this field and reconciler should be:
    1. apply machine annotation `machine.cluster.x-k8s.io/certificates-expiry` during etcd node creation.
    2. check if roll out is needed. Refer https://github.com/kubernetes-sigs/cluster-api/blob/main/controlplane/kubeadm/internal/controllers/controller.go#L380.
    3. if roll out is needed for any etcd node, upgrade all etcd nodes.

However, not all certificates in cp node would be renewed even both etcd and cp nodes have changed, since apiserver-etcd-client key pair is created by etcdadm controller (refer the sequence diagram) and it always stays the same. We need to fix this bug by changing [etcdadm controller’s code](https://github.com/aws/etcdadm-controller/blob/main/controllers/certs.go#L81), it should update apiserver-etcd-client secret when etcd nodes get rollout. 

### Cluster Yaml Spec Example

```
apiVersion: anywhere.eks.amazonaws.com/v1alpha1
kind: Cluster
spec:
  certificatesConfiguration:
    rolloutBeforeExpiry: 
      // whether to rollout etcd and cp nodes before certs expiring
      enabled: true/false (false by default, auto certs renew require opt-in) 
      // allow adding finer control in the future
```

### Tech Details

1. `certificatesConfiguration.rolloutBeforeExpiry.enabled` should be true when user run “eksctl anywhere generate clusterconfig”
2. When `rolloutBeforeExpiry` is enabled, the cp/etcd nodes are rollout when certs are expiring in **7** days
3. etcd certs renewal has a side-effect of renewing cp certs (etcd-rollout → etcd api change → cp rollout → cp certs renewal). We should only renew etcd certs to avoid unnecessary cp rollout.

## Alternative Solutions

### Cronjob

The script run by the job would need to be tailored/tested for each provider/OS combinations. It’s error prone and hard to develop/maintain. Manually renew certs in bottlerocket is very tricky. Instead of running the binary locally, you have to pull the etcdadm/kubeadm image locally, mount folders, configuring container runtime, calling bottlerocket api etc to make it work. Besides, when there is bug or dependencies change, you would perhap need to test the fix against multiple environments.

### Increase all certs expiring time to as long as the CA’s expiring time (10 years)

This would increase vulnerability.

## FAQs

Q: Would cp node lose connection with etcd nodes after etcd nodes were recreated to renew certificates? As the etcd node IPs can change.
A: Yes, cp nodes would lost connection if etcd node’s IP changed. But KCP (kubeadm-control-plane) would watch etcd endpoints, and roll out new cp nodes when it detects any etcd endpoint change.

Q: Do you have proof that apiserver-etcd-client key stays the same after etcd and cp nodes recreated
A: Yes. I have done experiment with CloudStack. The following graph shows that the apiserver-kube-client.crt has longer expiration time than the apiserver-etcd-client.crt.  And the apiserver-etcd-client.crt’s expiration time always stays the same as long as the secret doesn’t change.
Q: Should we fix the stale apiserver-etcd-client bug in kubeadm controller, aka, renew the cert in kubeadm controller?
A: 1) Ownership: apiserver-etcd-client cert is created by etcdadm controller and owned by it, it’s more etcdadm’s business logic to renew it. 2) Flexibility, etcdadm controller should work with any controlplane controller, not bundling with KCP controller. 3) Simplicity, refreshing apiserver-etcd-client cert can be done in a few lines in etcdadm controller’s health checker 4) Responsibility, EKSA team owns etcdadm controller but not kubeadm controller

Q: Does updating the secret that contains apiserver-etcd-client trigger new control plane nodes to rollout?
A: Secret was updated manually and no control-plane rollout has been observed.

Q: How to renew certs manually?
A: https://anywhere.eks.amazonaws.com/docs/tasks/cluster/manually-renew-certs/

Q: What impacts to customer deployed applications and kubeconfigs are there?
A: When the feature is added, there will be no impact to the existing clusters. The existing clusters will only be changed when the user explicitly enable the certs-auto-renewal feature.

Q: Are there any side effects/issues if a user does not bootstrap with `certificatesConfiguration.rolloutBeforeExpiry.enabled: true` .
A: No side effect when a user doesn’t explicit opt in auto-certs-renewal.