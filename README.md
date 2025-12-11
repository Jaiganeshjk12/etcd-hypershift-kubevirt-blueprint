# etcd-hypershift-kubevirt-blueprint
This is a Kanister blueprint to backup and restore etcd database for OCP clusters that runs inside another openshift cluster with hypershift and KubeVirt provider. 

This follows the backup and restore procedure for hosted controlplanes documented [here](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/hosted_control_planes/index#hcp-backup-restore-on-premise)

Hypershift clusters with Kubevirt basically runs the controlplane components in its own namespace within a management cluster. KubeVirt VMs are bootstrapped as worker nodes (https://hypershift.pages.dev/how-to/kubevirt/create-kubevirt-cluster/).

In such clusters, etcd is not accessible within the cluster to access and take snapshot of it. 

All the steps in this document has to be done from the management cluster where the hypershift clusters are hosted(where the controlplane components are run). 

This blueprint has backup, restore and delete action to manage the backup, restore and retire steps in Kasten respectively. 

## Backup

### Deploy the blueprint in Kasten namespace

Create the blueprint from this repository in your cluster. 
```bash
kubectl create -f https://raw.githubusercontent.com/Jaiganeshjk12/etcd-hypershift-kubevirt-blueprint/refs/heads/main/etcd-blueprint.yaml -n kasten-io
```
### Annotate the hostedcluster resource or create a blueprintbinding

Once the Blueprint is created, annotate the hostedcluster resource in the default clusters namespace(this can change depending on the ACM setup)
```bash
kubectl annotate hostedclusters.hypershift.openshift.io <hosted-cluster-name> -n <hosted-cluster-namespace> kanister.kasten.io/blueprint=etcd-blueprint-for-hosted-control-plane
```
Alternatively, You could also create a new blueprintbinding pointing to the hostedcluster resources in the cluster. This makes sure that Kasten always uses this blueprint when it backs up the hostedcluster CR. 
```
apiVersion: config.kio.kasten.io/v1alpha1
kind: BlueprintBinding
metadata:
  name: etcd-hosted-cluster-blueprint-binding
  namespace: kasten-io
spec:
  blueprintRef:
    name: etcd-blueprint-for-hosted-control-plane
    namespace: kasten-io
  resources:
    matchAll:
    - type:
        operator: In
        values:
        - name: ""
          resource: hostedclusters
```
### Create a policy in Kasten
Once the hostedcluster resource is annotated or blueprintbinding resource is created, Use Kasten to create policy for the hostedcluster namespace and setup location profile for Kanister action within the policy. 
No need to use filters for resources because we might also need some encryption keys that are used for the hypershift cluster etcd. 

## Restore
This blueprint expects the hostedcluster to be already present before the restore is started. During the disaster, recreate the hostedcluster and then run the restore from Kasten so that Kasten can be restored.
You can use restore from Kasten UI (https://docs.kasten.io/latest/usage/restore#restoring-existing-applications) and filter out the resources that needs to be restored. Make sure to select the hostedcluster which is the subject of this blueprint. 
Once the restore completes, Please validate the cluster API and resources within the hypershift hosted cluster by connecting to that cluster. 

## Limitations
- This blueprint assumes that the hostedcluster is already running before the restore starts.
- Restore works on the hostedcluster with the same name. 
