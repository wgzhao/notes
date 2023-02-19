# Migrate kubernetes cluster from Flannel to Calico

[Source](https://projectcalico.docs.tigera.io/getting-started/kubernetes/flannel/migration-from-flannel "Permalink to Migrate a Kubernetes cluster from flannel/Canal to Calico")

4 MINUTE READ 

### Big picture

Migrate an existing Kubernetes cluster with flannel/Canal to Calico networking.

### Value

If you are already using flannel for networking, it is easy to migrate to Calico’s native VXLAN networking. Calico VXLAN is fully equivalent to flannel vxlan, but you get the benefits of the broader range of features offered by Calico with an active maintainer community.

### Concepts

#### Limitations of host-local IPAM in flannel

Flannel networking uses the host-local IPAM (IP address management) CNI plugin, which provides simple IP address management for your cluster. Although simple, it has limitations:

* When you create a node, it is pre-allocated a CIDR. If the number of pods per-node exceeds the number of IP addresses available per node, you must recreate the cluster. Conversely, if the number of pods is much smaller than the number of addresses available per node, IP address space is not efficiently used; as you scale out and IP addresses are depleted, inefficiencies become a pain point.
* Because each node has a pre-allocated CIDR, pods must always have an IP address assigned based on the node it is running on. Being able to allocate IP addresses based on other attributes (for example, the pod’s namespace), provides flexiblity to meet use cases that arise.

Migrating to Calico IPAM solves these use cases and more. For advantages of Calico IPAM, see [Blog: Live Migration from Flannel to Calico](https://www.projectcalico.org/live-migration-from-flannel-to-calico/).

#### Methods for migrating to Calico networking

There are two ways to switch your cluster to use Calico networking. Both methods give you a fully-functional Calico cluster using VXLAN networking between pods.

* **Create a new cluster using Calico and migrate existing workloads**

  If you have the ability to migrate worloads from one cluster to the next without caring about downtime, this is the easiest method: [create a new cluster using Calico](https://projectcalico.docs.tigera.io/getting-started/kubernetes/quickstart).
* **Live migration on an existing cluster**

  If your workloads are already in production, or downtime is not an option, use the live migration tool that performs a rolling update of each node in the cluster.

### Before you begin…

**Required**

* A cluster with flannel for networking using the VXLAN backend.
* Flannel version v0.9.1 or higher (Canal version v3.7.0 or greater).
* Flannel must have been installed using a **Kubernetes daemon set** and configured: 
  * To use the Kubernetes API for storing its configuration (as opposed to etcd)
  * With `DirectRouting` disabled (default)
* Cluster must allow for: 
  * Adding/deleting/modifying node labels
  * Modifying and deleting of the flannel daemon set. For example, it must not be installed using the Kubernetes Addon-manager.

### How to

* [Migrate from flannel networking to Calico networking, live migration](https://projectcalico.docs.tigera.io/getting-started/kubernetes/flannel/migration-from-flannel#migrate-from-flannel-networking-to-calico-networking-live-migration)
* [Modify flannel configuration](https://projectcalico.docs.tigera.io/getting-started/kubernetes/flannel/migration-from-flannel#modify-flannel-configuration)
* [View migration status](https://projectcalico.docs.tigera.io/getting-started/kubernetes/flannel/migration-from-flannel#view-migration-status)
* [View migration logs](https://projectcalico.docs.tigera.io/getting-started/kubernetes/flannel/migration-from-flannel#view-migration-logs)
* [Revert migration](https://projectcalico.docs.tigera.io/getting-started/kubernetes/flannel/migration-from-flannel#revert-migration)

#### Migrate from flannel networking to Calico networking, live migration

1. Install Calico.

      kubectl apply -f https://docs.projectcalico.org/manifests/flannel-migration/calico.yaml
2. Start the migration controller.

      kubectl apply -f https://docs.projectcalico.org/manifests/flannel-migration/migration-job.yaml

  You will see nodes begin to update one at a time.
3. Monitor the migration.

      kubectl get jobs -n kube-system flannel-migration

  When the host node is upgraded, the migration controller may be rescheduled several times. The installation is complete when the output of the above command shows 1/1 completions. For example:

      NAME                COMPLETIONS   DURATION   AGE
    flannel-migration   1/1           2m59s      5m9s
4. Delete the migration controller.

      kubectl delete -f https://docs.projectcalico.org/manifests/flannel-migration/migration-job.yaml

#### Modify flannel configuration

The migration controller autodetects your flannel configuration, and in most cases, does not require additional configuration. If you require special configuration, the migration tool provides the following options, which can be set as environment variables within the pod.

Configuration optionsDescriptionDefaultFLANNEL\_NETWORKIPv4 network CIDR used by flannel for the cluster.Automatically detectedFLANNEL\_DAEMONSET\_NAMEName of the flannel daemon set in the kube-system namespace.kube-flannel-ds-amd64FLANNEL\_MTUMTU for the flannel VXLAN device.Automatically detectedFLANNEL\_IP\_MASQWhether masquerading is enabled for outbound traffic.Automatically detectedFLANNEL\_SUBNET\_LENPer-node subnet length used by flannel.24FLANNEL\_ANNOTATION\_PREFIXValue provided via the kube-annotation-prefix option to flannel.flannel.alpha.coreos.comFLANNEL\_VNIThe VNI used for the flannel network.1FLANNEL\_PORTUDP port used for VXLAN.8472CALICO\_DAEMONSET\_NAMEName of the calico daemon set in the kube-system namespace.calico-nodeCNI\_CONFIG\_DIRFull path on the host in which to search for CNI config files./etc/cni/net.d

#### View migration status

View the controller’s current status.

    kubectl get pods -n kube-system -l k8s-app=flannel-migration-controller

#### View migration logs

View migration logs to see if any actions are required.

    kubectl logs -n kube-system -l k8s-app=flannel-migration-controller

#### Revert migration

If you need to revert a cluster from Calico back to flannel, follow these steps.

1. Remove the migration controller and Calico.

      kubectl delete -f https://docs.projectcalico.org/manifests/flannel-migration/migration-job.yaml
    kubectl delete -f https://docs.projectcalico.org/manifests/flannel-migration/calico.yaml
2. Determine the nodes that were migrated to Calico.

      kubectl get nodes -l projectcalico.org/node-network-during-migration=calico

Then, for each node found above, run the following commands to delete Calico.

1. Cordon and drain the node.

      kubectl drain <node name>
2. Log in to the node and remove the CNI configuration.

      rm /etc/cni/net.d/10-calico.conflist
3. Reboot the node.
4. Enable flannel on the node.

      kubectl label node <node name> projectcalico.org/node-network-during-migration=flannel --overwrite
5. Uncordon the node.

      kubectl uncordon <node name>

After the above steps have been completed on each node, perform the following steps.

1. Remove the `nodeSelector` from the flannel daemonset.

      kubectl patch ds/kube-flannel-ds-amd64 -n kube-system -p '{"spec": {"template": {"spec": {"nodeSelector": null}}}}'
2. Remove the migration label from all nodes.

      kubectl label node --all projectcalico.org/node-network-during-migration-

### Next steps

Learn about [Calico IP address management](https://projectcalico.docs.tigera.io/networking/ipam)

[Slack](https://slack.projectcalico.org/)[Discourse](https://discuss.projectcalico.org/)[GitHub](https://github.com/projectcalico/calico)[Twitter](https://twitter.com/projectcalico)[YouTube](https://www.youtube.com/channel/UCFpTnXDNcBoXI4gqCDmegFA)[Free Online Training](https://www.tigera.io/events/)