# Deploy the AKS Clusters in two different regions

Now that you [generated your client-facing and AKS ingress controller TLS certificates](./05-ca-certificates.md), the next step in the [AKS baseline multi cluster reference implementation](/README.md) is deploying the AKS clusters and its adjacent Azure resources.

## Expected results

Following the steps below will result in the provisioning of the AKS multi cluster solution.

| Object                         | Purpose                                                                                                                            |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| GitHub Workflow                | A GitHub Workflow that deploys the multi cluster infrastructure in two different regions.                                          |
| Two Azure Application Gateways | One Azure Application Gateway in East US 2 and another one from its paired region Central US.                                      |
| Two configured AKS Clusters    | Two AKS Clusters in the identified regions, both bootstrapped with their initial desired state. Flux, workload identity, and more. |

## Steps

> :book: The selected locations are `East US 2` and `Central US` which are Azure paired regions. All in all, the team resolution is to have a single CI/CD pipeline that is aware of the multiple clusters being deployed, and could take measures in case of errors while deploying for a particular region. This pipeline uses a common stamp for the cluster creation with different parameter files per region, and it will be also enrolling the AKS Clusters in GitOps to help with the initial desired state. From there, Flux will take care of the AKS cluster bootstrapping process. In this case, the app team decided to use GitHub Actions. They are going to create a workflow grouping all AKS clusters in different regions that are serving the same application. They know that every change in their cluster stamp or workflow will impact most probably all clusters. But there are some special cases they want to contemplate like adding a new cluster to their fleet. For those scenarios, they are tagging the pipeline execution to exclude and remain them untouched from that particular execution.

![The AKS Clusters deployment diagram depicting the proposed cluster fleet topology running from different regions.](./images/aks-cluster-mgmnt.png)

> :bulb: Multi Cluster and Federation's repos could be a monorepo or multiple repos as displayed from the digram above. In this reference implementation, the workload manifests, and ARM templates are shipped together from a single repo.

> :bulb: Another interesting use case that this architecture could help with is when AKS introduces _Preview Features_ in the same or different regions. They could in some case be a breaking in an upcoming major releases like happened with `containerd` as the new default runtime. In those situtations, you might want to do some A/B testing without fully disrupting your live and stable AKS cluster.

1. Obtain shared services resource details

    ```bash
    LOGANALYTICSWORKSPACEID=$(az deployment group show -g rg-bu0001a0042-shared -n shared-svcs-stamp --query properties.outputs.logAnalyticsWorkspaceId.value -o tsv)
    CONTAINERREGISTRYID=$(az deployment group show -g rg-bu0001a0042-shared -n shared-svcs-stamp --query properties.outputs.containerRegistryId.value -o tsv)
    export ACR_NAME_AKS_MRB=$(az deployment group show -g rg-bu0001a0042-shared -n shared-svcs-stamp --query properties.outputs.containerRegistryName.value -o tsv)
    echo LOGANALYTICSWORKSPACEID: $LOGANALYTICSWORKSPACEID
    echo CONTAINERREGISTRYID: $CONTAINERREGISTRYID
    echo ACR_NAME_AKS_MRB: $ACR_NAME_AKS_MRB
    ```

1. Upload images to your Azure Container Registry that are referenced bootstrapping.

    ```bash
    az acr import --source docker.io/weaveworks/kured:1.14.0 -n $ACR_NAME_AKS_MRB --force
    az acr import --source docker.io/library/traefik:v2.10.4 -n $ACR_NAME_AKS_MRB --force
    ```

1. Get the corresponding AKS cluster spoke VNet resource IDs for the app team working on the application A0042.

    > :book: The app team will be deploying to a spoke VNet, that was already provisioned by the network team.

    ```bash
    RESOURCEID_VNET_BU0001A0042_03=$(az deployment group show -g rg-enterprise-networking-spokes -n spoke-BU0001A0042-03 --query properties.outputs.clusterVnetResourceId.value -o tsv)
    RESOURCEID_VNET_BU0001A0042_04=$(az deployment group show -g rg-enterprise-networking-spokes -n spoke-BU0001A0042-04 --query properties.outputs.clusterVnetResourceId.value -o tsv)
    echo RESOURCEID_VNET_BU0001A0042_03: $RESOURCEID_VNET_BU0001A0042_03
    echo RESOURCEID_VNET_BU0001A0042_04: $RESOURCEID_VNET_BU0001A0042_04
    ```

    1. Assign required permissions to the [GitHub workflow's](https://learn.microsoft.com/azure/developer/github/connect-from-azure) managed (federated) identity.

        ```bash
        # Get the managed identity to assign necessary deployment permissions
        export GITHUB_FEDERATED_IDENTITY_PRINCIPALID=$(az deployment group show -g rg-bu0001a0042-shared -n shared-svcs-stamp --query 'properties.outputs.githubFederatedIdentityPrincipalId.value' -o tsv)
        echo GITHUB_FEDERATED_IDENTITY_PRINCIPALID: $GITHUB_FEDERATED_IDENTITY_PRINCIPALID

        # Assign built-in Contributor RBAC role for creating resource groups and performing deployments at subscription level
        az role assignment create --assignee $GITHUB_FEDERATED_IDENTITY_PRINCIPALID --role 'Contributor'

        # Assign built-in User Access Administrator RBAC role since granting RBAC access to other resources during the cluster creation will be required at subscription level (e.g. AKS-managed Internal Load Balancer, ACR, Managed Identities, etc.)
        az role assignment create --assignee $GITHUB_FEDERATED_IDENTITY_PRINCIPALID --role 'User Access Administrator'
        ```

    1. Create federated identity secrets in your GitHub repository.

        ```bash
        export GITHUB_FEDERATED_IDENTITY_CLIENTID=$(az deployment group show -g rg-bu0001a0042-shared -n shared-svcs-stamp --query 'properties.outputs.githubFederatedIdentityClientId.value' -o tsv)
        echo GITHUB_FEDERATED_IDENTITY_CLIENTID: $GITHUB_FEDERATED_IDENTITY_CLIENTID

        export SUBSCRIPTION_ID=$(az account show --query id -o tsv)
        echo SUBSCRIPTION_ID: $SUBSCRIPTION_ID
        
        gh secret set AZURE_CLIENT_ID -b"${GITHUB_FEDERATED_IDENTITY_CLIENTID}"  --repo $GITHUB_USER_NAME_AKS_MRB/aks-baseline-multi-region
        gh secret set AZURE_TENANT_ID -b"${TENANTID_AZURERBAC_AKS_MRB}"  --repo $GITHUB_USER_NAME_AKS_MRB/aks-baseline-multi-region
        gh secret set AZURE_SUBSCRIPTION_ID  -b"${SUBSCRIPTION_ID}"  --repo $GITHUB_USER_NAME_AKS_MRB/aks-baseline-multi-region
        ```

    1. Create `APP_GATEWAY_LISTENER_REGION1_CERTIFICATE_BASE64` and `APP_GATEWAY_LISTENER_REGION2_CERTIFICATE_BASE64` secret in your GitHub repository.

        ```bash
        gh secret set APP_GATEWAY_LISTENER_REGION1_CERTIFICATE_BASE64  -b"${APP_GATEWAY_LISTENER_REGION1_CERTIFICATE_BASE64_AKS_MRB}" --repo $GITHUB_USER_NAME_AKS_MRB/aks-baseline-multi-region
        gh secret set APP_GATEWAY_LISTENER_REGION2_CERTIFICATE_BASE64  -b"${APP_GATEWAY_LISTENER_REGION2_CERTIFICATE_BASE64_AKS_MRB}" --repo $GITHUB_USER_NAME_AKS_MRB/aks-baseline-multi-region
        ```

    1. Create `AKS_INGRESS_CONTROLLER_CERTIFICATE_BASE64` secret in your GitHub repository.

        ```bash
        gh secret set AKS_INGRESS_CONTROLLER_CERTIFICATE_BASE64 -b"${AKS_INGRESS_CONTROLLER_CERTIFICATE_BASE64_AKS_MRB}" --repo $GITHUB_USER_NAME_AKS_MRB/aks-baseline-multi-region
        ```

    1. Copy the GitHub workflow file into the expected directory.

        ```bash
        mkdir -p .github/workflows
        cat github-workflow/aks-deploy.yaml > .github/workflows/aks-deploy.yaml
        ```

    1. Generate cluster parameter file per region.

        Verify all variables are populated:

        ```bash
        echo RESOURCEID_VNET_BU0001A0042_03: $RESOURCEID_VNET_BU0001A0042_03 && \
        echo RESOURCEID_VNET_BU0001A0042_04: $RESOURCEID_VNET_BU0001A0042_04 && \
        echo TENANTID_K8SRBAC_AKS_MRB: $TENANTID_K8SRBAC_AKS_MRB && \
        echo AADOBJECTID_GROUP_CLUSTERADMIN_BU0001A004203_AKS_MRB: $AADOBJECTID_GROUP_CLUSTERADMIN_BU0001A004203_AKS_MRB && \
        echo AADOBJECTID_GROUP_CLUSTERADMIN_BU0001A004204_AKS_MRB: $AADOBJECTID_GROUP_CLUSTERADMIN_BU0001A004204_AKS_MRB && \
        echo LOGANALYTICSWORKSPACEID: $LOGANALYTICSWORKSPACEID && \
        echo CONTAINERREGISTRYID: $CONTAINERREGISTRYID && \
        echo GITHUB_USER_NAME_AKS_MRB: $GITHUB_USER_NAME_AKS_MRB
        ```

        Update each region's cluster parameter file:

        ```bash
        # Region 1
        sed -i "s#<cluster-spoke-vnet-resource-id>#${RESOURCEID_VNET_BU0001A0042_03}#g" ./azuredeploy.parameters.eastus2.json && \
        sed -i "s#<tenant-id-with-user-admin-permissions>#${TENANTID_K8SRBAC_AKS_MRB}#g" ./azuredeploy.parameters.eastus2.json && \
        sed -i "s#<azure-ad-aks-admin-group-object-id>#${AADOBJECTID_GROUP_CLUSTERADMIN_BU0001A004203_AKS_MRB}#g" ./azuredeploy.parameters.eastus2.json && \
        sed -i "s#<log-analytics-workspace-id>#${LOGANALYTICSWORKSPACEID}#g" ./azuredeploy.parameters.eastus2.json && \
        sed -i "s#<container-registry-id>#${CONTAINERREGISTRYID}#g" ./azuredeploy.parameters.eastus2.json && \
        sed -i "s#<your-github-org>#${GITHUB_USER_NAME_AKS_MRB}#g" ./azuredeploy.parameters.eastus2.json

        # Region 2
        sed -i "s#<cluster-spoke-vnet-resource-id>#${RESOURCEID_VNET_BU0001A0042_04}#g" ./azuredeploy.parameters.centralus.json && \
        sed -i "s#<tenant-id-with-user-admin-permissions>#${TENANTID_K8SRBAC_AKS_MRB}#g" ./azuredeploy.parameters.centralus.json && \
        sed -i "s#<azure-ad-aks-admin-group-object-id>#${AADOBJECTID_GROUP_CLUSTERADMIN_BU0001A004204_AKS_MRB}#g" ./azuredeploy.parameters.centralus.json && \
        sed -i "s#<log-analytics-workspace-id>#${LOGANALYTICSWORKSPACEID}#g" ./azuredeploy.parameters.centralus.json && \
        sed -i "s#<container-registry-id>#${CONTAINERREGISTRYID}#g" ./azuredeploy.parameters.centralus.json && \
        sed -i "s#<your-github-org>#${GITHUB_USER_NAME_AKS_MRB}#g" ./azuredeploy.parameters.centralus.json
        ```

    1. Customize your GitOps manifests to pull images from your private ACR.

        ```bash
        find . -type f -name "kustomization.yaml" -exec sed -i "s/REPLACE_ME_WITH_YOUR_ACRNAME/${ACR_NAME_AKS_MRB}/" {} +
        ```

    1. The workflow is triggered when a push on the `main` branch is detected. Therefore, push the changes to your forked repo.

        > :book: The app team monitors the workflow execution as this is impacting a critical piece of infrastructure. This flow works for both new or existing AKS clusters. The workflow deploys the multiple clusters in different regions, and configures the desired state for them.

        ```bash
        git add -u && git add .github/workflows/aks-deploy.yaml && git commit -m "Customize manifests and setup GitHub CD workflow" && git push origin main
        ```

        > :bulb: You might want to convert this GitHub workflow into a template since your organization or team might need to handle multiple AKS clusters. For more information, please take a look at [Sharing Workflow Templates within your organization](https://docs.github.com/actions/configuring-and-managing-workflows/sharing-workflow-templates-within-your-organization).

    1. You can continue only after the GitHub Workflow completes successfully.

        ```bash
        until export GH_WF_STATUS=$(gh api /repos/:owner/:repo/actions/runs/$(gh api /repos/:owner/:repo/actions/runs -q ".workflow_runs[0].id") -q ".status" 2> /dev/null) && [[ $GH_WF_STATUS == "completed" ]]; do echo "Monitoring GitHub workflow execution: ${GH_WF_STATUS}" && sleep 20; done
        ```

    1. Get the cluster names for regions 1 and 2.

        ```bash
        export AKS_CLUSTER_NAME_BU0001A0042_03_AKS_MRB=$(az deployment group show -g rg-bu0001a0042-03 -n cluster-stamp --query properties.outputs.aksClusterName.value -o tsv)
        export AKS_CLUSTER_NAME_BU0001A0042_04_AKS_MRB=$(az deployment group show -g rg-bu0001a0042-04 -n cluster-stamp --query properties.outputs.aksClusterName.value -o tsv)
        echo AKS_CLUSTER_NAME_BU0001A0042_03_AKS_MRB: $AKS_CLUSTER_NAME_BU0001A0042_03_AKS_MRB
        echo AKS_CLUSTER_NAME_BU0001A0042_04_AKS_MRB: $AKS_CLUSTER_NAME_BU0001A0042_04_AKS_MRB
        ```

    1. Install kubectl 1.24 or newer. (kubectl supports ±1 Kubernetes version.)

       ```bash
       sudo az aks install-cli
       kubectl version --client 
       ```

    1. Get AKS `kubectl` credentials.

        > In the [Azure Active Directory Integration](03-aad.md) step, we placed our cluster under AAD group-backed RBAC. This is the first time we are seeing this used. `az aks get-credentials` allows you to use `kubectl` commands against your cluster. Without the AAD integration, you'd have to use `--admin` here, which isn't what we want to happen. In a following step, you'll log in with a user that has been added to the Azure AD security group used to back the Kubernetes RBAC admin role. Executing the first `kubectl` command below will invoke the AAD login process to auth the _user of your choice_, which will then be checked against Kubernetes RBAC to perform the action. The user you choose to log in with _must be a member of the AAD group bound_ to the `cluster-admin` ClusterRole. For simplicity you could either use the "break-glass" admin user created in [Azure Active Directory Integration](03-aad.md) (`bu0001a0042-admin`) or any user you assigned to the `cluster-admin` group assignment in your [`cluster-rbac.yaml`](cluster-manifests/cluster-rbac.yaml) file. If you skipped those steps you can use `--admin` to proceed, but proper AAD group-based RBAC access is a critical security function that you should invest time in setting up.

        ```bash
        az aks get-credentials -g rg-bu0001a0042-03 -n $AKS_CLUSTER_NAME_BU0001A0042_03_AKS_MRB
        az aks get-credentials -g rg-bu0001a0042-04 -n $AKS_CLUSTER_NAME_BU0001A0042_04_AKS_MRB
        ```

        :warning: At this point two important steps are happening:

        - The `az aks get-credentials` command will be fetch a `kubeconfig` containing references to the AKS cluster you have created earlier.
        - To _actually_ use the cluster you will need to authenticate. For that, run any `kubectl` commands which at this stage will prompt you to authenticate against Azure Active Directory. For example, run the following command:

        ```bash
        kubectl get nodes --context $AKS_CLUSTER_NAME_BU0001A0042_03_AKS_MRB
        kubectl get nodes --context $AKS_CLUSTER_NAME_BU0001A0042_04_AKS_MRB
        ```

        Once the authentication happens successfully, some new items will be added to your `kubeconfig` file such as an `access-token` with an expiration period. For more information on how this process works in Kubernetes please refer to [the related documentation](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens).

    1. Ensure Flux in region 1 and 2 has created the workload namespaces.

        :bulb: Please notice that both namespaces are Kustomization overlays, and as such they were customized to be annotated with the region number.

        ```bash
        # press Ctrl-C once you receive a successful response
        kubectl get ns a0042 -o yaml -w --context $AKS_CLUSTER_NAME_BU0001A0042_03_AKS_MRB

        # press Ctrl-C once you receive a successful response
        kubectl get ns a0042 -o yaml -w --context $AKS_CLUSTER_NAME_BU0001A0042_04_AKS_MRB
        ```

### Save your work in-progress

```bash
# run the saveenv.sh script at any time to save environment variables created above to aks_baseline.env
./saveenv.sh

# if your terminal session gets reset, you can source the file to reload the environment variables
# source aks_baseline.env
```

### Next step

:arrow_forward: [Prepare for the workload by installing its prerequisites](./07-workload-prerequisites.md)
