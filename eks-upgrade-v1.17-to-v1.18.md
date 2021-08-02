# VFS EKS Cluster upgrade v1.17 to v1.18




### Pre-Upgrade Checklist:

1. Compare the Kubernetes version of your cluster control plane to the Kubernetes version of your nodes\.
   + Get the Kubernetes version of your cluster control plane with the following command\.

     ```
     kubectl version --short
     ```
   + Get the Kubernetes version of your nodes with the following command\. This command returns all self\-managed and managed Amazon EC2 and Fargate nodes\. Each Fargate pod is listed as its own node\.

     ```
     kubectl get nodes
     ```

   The Kubernetes minor version of the managed and Fargate nodes in your cluster must be the same as the version of your control plane's current version before you update your control plane to a new Kubernetes version\.
1. The pod security policy admission controller is enabled by default on Amazon EKS clusters\. Before updating your cluster, ensure that the proper pod security policies are in place before you update to avoid any issues\. You can check for the default policy with the following command:

   ```
   kubectl get psp eks.privileged
   ```

   If you receive the following error, see [default pod security policy](https://docs.aws.amazon.com/eks/latest/userguide/pod-security-policy.html#default-psp) before proceeding\.

   ```
   Error from server (NotFound): podsecuritypolicies.extensions "eks.privileged" not found
   ```

1. If you originally deployed your cluster on Kubernetes `1.17` or earlier, then you may need to remove a discontinued term from your CoreDNS manifest\.

   1. Check to see if your CoreDNS manifest has a line that only has the word `upstream`\.

      ```
      kubectl get configmap coredns -n kube-system -o jsonpath='{$.data.Corefile}' | grep upstream
      ```

      If no output is returned, your manifest doesn't have the line and you can skip to the next step to update your cluster\. If the word `upstream` is returned, then you need to remove the line\.

   1. Edit the configmap, removing the line near the top of the file that only has the word `upstream`\. Don't change anything else in the file\. After the line is removed, save the changes\.

      ```
      kubectl edit configmap coredns -n kube-system -o yaml
      ```
### Upgrade Process:
------
#### Control Plane/ Master Node
##### [ AWS Management Console ]

   1. Open the Amazon EKS console at [https://console\.aws\.amazon\.com/eks/home\#/clusters](https://console.aws.amazon.com/eks/home#/clusters)\.

   1. Choose the name of the cluster to update and choose **Update cluster version**\.

   1. For **Kubernetes version**, select the version to update your cluster to and choose **Update**\.

   1. For **Cluster name**, type the name of your cluster and choose **Confirm**\.

      The update takes several minutes to complete\.

------
##### [ AWS CLI ]

   1. Update your cluster with the following AWS CLI command\. Replace the *`<example-values>`* \(including *`<>`*\) with your own\.

      ```
      aws eks update-cluster-version \
       --region <region-code> \
       --name <my-cluster> \
       --kubernetes-version <1.21>
      ```

      Output:

      ```
      {
          "update": {
              "id": "<b5f0ba18-9a87-4450-b5a0-825e6e84496f>",
              "status": "InProgress",
              "type": "VersionUpdate",
              "params": [
                  {
                      "type": "Version",
                      "value": "1.21"
                  },
                  {
                      "type": "PlatformVersion",
                      "value": "eks.1"
                  }
              ],
      ...
              "errors": []
          }
      }
      ```

   1. Monitor the status of your cluster update with the following command\. Use the cluster name and update ID that the previous command returned\. Your update is complete when the status appears as `Successful`\. The update takes several minutes to complete\.

      ```
      aws eks describe-update \
        --region <region-code> \
        --name <my-cluster> \
        --update-id <b5f0ba18-9a87-4450-b5a0-825e6e84496f>
      ```

      Output:

      ```
      {
          "update": {
              "id": "b5f0ba18-9a87-4450-b5a0-825e6e84496f",
              "status": "Successful",
              "type": "VersionUpdate",
              "params": [
                  {
                      "type": "Version",
                      "value": "1.21"
                  },
                  {
                      "type": "PlatformVersion",
                      "value": "eks.1"
                  }
              ],
      ...
              "errors": []
          }
      }
      ```
#### Managed worker node group
##### [ AWS Management Console ]

**To update a node group version with the AWS Management Console**

1. Open the Amazon EKS console at [https://console\.aws\.amazon\.com/eks/home\#/clusters](https://console.aws.amazon.com/eks/home#/clusters)\.

1. Choose the cluster that contains the node group to update\.

1. If at least one node group has an available update, a box appears at the top of the page notifying you of the available update\. If you select the **Configuration** tab and then the **Compute** tab, you'll see **Update now** in the **AMI release version** column in the **Node Groups** table for the node group that has an available update\. To update the node group, select **Update now**\. You won't see a notification for node groups that were deployed with a custom AMI\. 
1. On the **Update Node Group version** page, select:
   + **Update Node Group version** – This option is unavailable if you deployed a custom AMI or your Amazon EKS optimized AMI is currently on the latest version for your cluster\.


1. For **Update strategy**, select one of the following options and then choose **Update**\.
   + **Rolling update** – This option respects the pod disruption budgets for your cluster\. Updates fail if there is a pod disruption budget issue that causes Amazon EKS to be unable to gracefully drain the pods that are running on this node group\.
   + **Force update** – This option doesn't respect pod disruption budgets\. Updates occur regardless of pod disruption budget issues by forcing node restarts to occur\.
------

### Post Upgrade Checklist:

 #### Updating the `kube-proxy` add\-on manually


If you have a 1\.17 or earlier cluster, after the cluster and managed nodegroup update is done, complete the following steps to manually update the **kube-proxy** add\-on\. 

**Important**  
Update your cluster and nodes to a new Kubernetes minor version before updating `kube-proxy` to the same minor version as your updated cluster's minor version\.

1. Check the current version of your `kube-proxy` deployment\.

   ```
   kubectl get daemonset kube-proxy --namespace kube-system -o=jsonpath='{$.spec.template.spec.containers[:1].image}'
   ```

   Example output

   ```
   602401143452.dkr.ecr.eu-west-2.amazonaws.com/eks/kube-proxy:v1.17.9-eksbuild.1
   ```

1. Replace `1.17.9-eksbuild.1` with the `1.18.8-eksbuild.1` version listed in the kube-proxy version deployed with each Amazon EKS supported cluster version table for your cluster version\.

   ```
   kubectl set image daemonset.apps/kube-proxy \
        -n kube-system \
        kube-proxy=602401143452.dkr.ecr.eu-west-2.amazonaws.com/eks/kube-proxy:v1.18.8-eksbuild.1
   ```
 1. Verify the current version of your `kube-proxy` deployment\.

   ```
   kubectl get daemonset kube-proxy --namespace kube-system -o=jsonpath='{$.spec.template.spec.containers[:1].image}'
   ```
   
   #### Updating the `core-dns` add\-on manually
   If you have a 1\.17 or earlier cluster, complete the following steps to update the add\-on\.

**To update the CoreDNS add\-on**

1. Check the current version of your CoreDNS deployment\.

   ```
   kubectl describe deployment coredns \
       --namespace kube-system \
       | grep Image \
       | cut -d "/" -f 3
   ```

   Output:

   ```
   coredns:v1.6.6-eksbuild.1
   ```
  1. If you originally deployed your cluster on Kubernetes `1.17` or earlier, then you may need to remove a discontinued term from your CoreDNS manifest\.
**Important**  
You must complete this before upgrading to CoreDNS version `1.7.0`, but it's recommended that you complete this step even if you're upgrading to an earlier version\. 

	   1. Check to see if your CoreDNS manifest has the line\.

	      ```
	      kubectl get configmap coredns -n kube-system -o jsonpath='{$.data.Corefile}' | grep upstream
	      ```

	      If no output is returned, your manifest doesn't have the line and you can skip to the next step to update CoreDNS\. If output is returned, then you need to remove the line\.

	   1. Edit the configmap with the following command, removing the line in the file that has the word `upstream` in it\. Do not change anything else in the file\. Once the line is removed, save the changes\.

	      ```
	      kubectl edit configmap coredns -n kube-system -o yaml
	      ```

1. Retrieve your current `coredns` image:

   ```
   kubectl get deployment coredns --namespace kube-system -o=jsonpath='{$.spec.template.spec.containers[:1].image}'
2. Update CoreDNS by replacing  `v1.6.6-eksbuild.1` with `1.7.0-eksbuild.1`
   ```
   kubectl set image --namespace kube-system deployment.apps/coredns \
       coredns=602401143452.dkr.ecr.eu-west-2.amazonaws.com/eks/coredns:v1.7.0-eksbuild.1
### Modify Online and Backoffice Ingress Objects(Optional)

In Kubernetes 1.18, there are two significant additions to Ingress: A new  `pathType`  field and a new  `IngressClass`  resource. The  `pathType`  field allows specifying how paths should be matched. In addition to the default  `ImplementationSpecific`  type, there are new  `Exact`  and  `Prefix`  path types.

The  `IngressClass`  resource is used to describe a type of Ingress within a Kubernetes cluster. Ingresses can specify the class they are associated with by using a new  `ingressClassName`  field on Ingresses. This new resource and field replace the deprecated  `kubernetes.io/ingress.class`  annotation.

Read More: https://kubernetes.io/docs/concepts/services-networking/ingress/#deprecated-annotation

**Note:** This isn't a breaking change in Kubernetes v1.18. Hence this step is optional.
