## Installing an OpenShift 4.5 cluster on Azure via UPI, using ARM templates

REF: https://docs.openshift.com/container-platform/4.5/installing/installing_azure/installing-azure-user-infra.html

Login to Azure = az login (account must have access to create objects in the resource group)
Create Service Principal Acct (contributor for the resource group)
Create a Storage Account

#### BEGIN SETUP

1. Generate the Install Configuration for Azure \
`openshift-install create install-config --dir=${INSTALL_DIR}`

2. Set Environment Variables 
    ```
    export CLUSTER_NAME=<CLUSTER_NAME> 
    export AZURE_REGION=<LOCATION> 
    export RESOURCE_GROUP=<AZURE_RG>
    export BASE_DOMAIN=<DOMAIN_NAME> 
    export STORAGESA=<STORAGE_ACC_NAME>
    export INSTALL_DIR=</path/to/install> 
    export BASE_DOMAIN_RESOURCE_GROUP=<BASE_DOMAIN_RG> 
    export KUBECONFIG=${INSTALL_DIR}/auth/kubeconfig 
    export SSH_KEY="ssh-rsa ..."
    ```
3. Create the Manifest files \
`openshift-install create manifests --dir=${INSTALL_DIR}`

4. Remove the template machine configs \
`rm -f ${INSTALL_DIR}/openshift/99_openshift-cluster-api_master-machines-*.yaml` \
`rm -f ${INSTALL_DIR}/openshift/99_openshift-cluster-api_worker-machineset-*.yaml`

5. Replace self-generated resource-group with target resource-group in \
`vi  ${INSTALL_DIR}/manifests/{cluster-dns-02-config.yml,cloud-provider-config.yaml}`

6. Record the infrastructure id and verify resource groups in \
`cat ${INSTALL_DIR}/manifests/cluster-infrastructure-02-config.yml`

7. Set the INFRA_ID environment variable \
`export INFRA_ID=<INFRA>`

8. Create IGNITION files \
`openshift-install create ignition-configs --dir=${INSTALL_DIR}`

9. Create the Azure Managed Identity and Set RBAC \
`az identity create -g ${RESOURCE_GROUP} -n ${INFRA_ID}-identity`
    ```
    export PRINCIPAL_ID=`az identity show -g ${RESOURCE_GROUP} -n ${INFRA_ID}-identity --query principalId --out tsv`

    export RESOURCE_GROUP_ID=`az group show -g ${RESOURCE_GROUP} --query id --out tsv`
    ```
    `az role assignment create --assignee "${PRINCIPAL_ID}" --role 'Contributor' --scope "${RESOURCE_GROUP_ID}"`

10. Upload RHCOS image and BOOTSTRAP.IGN file \
`az storage account create -g ${RESOURCE_GROUP} --location ${AZURE_REGION} --name ${STORAGESA} --kind Storage --sku Standard_LRS`
    ```
    export ACCOUNT_KEY=`az storage account keys list -g ${RESOURCE_GROUP} --account-name ${STORAGESA} --query "[0].value" -o tsv`

    export VHD_URL=`curl -s https://raw.githubusercontent.com/openshift/installer/release-4.5/data/data/rhcos.json | jq -r .azure.url`
    ```
    `az storage container create --name vhd --account-name ${STORAGESA} --account-key ${ACCOUNT_KEY}` \
    `az storage blob copy start --account-name ${STORAGESA} --account-key ${ACCOUNT_KEY} --destination-blob "rhcos.vhd" --destination-container vhd --source-uri "${VHD_URL}"`

    Note: Wait for upload to complete:
    ```
    status="unknown"
    while [ "$status" != "success" ]
    do
      status=`az storage blob show --container-name vhd --name "rhcos.vhd" --account-name ${STORAGESA} --account-key ${ACCOUNT_KEY} -o tsv --query properties.copy.status`
      echo $status
    done
    ```
11. Transfer the bootstrap ignition file to an Azure storage container \
`az storage container create --name files --account-name ${STORAGESA} --account-key ${ACCOUNT_KEY} --public-access blob` \
`az storage blob upload --account-name ${STORAGESA} --account-key ${ACCOUNT_KEY} -c "files" -f "${INSTALL_DIR}/bootstrap.ign" -n "bootstrap.ign"`

12. Create a new PRIVATE-DNS zone \
`az network private-dns zone create -g ${RESOURCE_GROUP} -n ${CLUSTER_NAME}.${BASE_DOMAIN}`

13. Setup VNET \
`az deployment group create -g ${RESOURCE_GROUP} --template-file "01_vnet.json" --parameters baseName="${INFRA_ID}"` \
`az network private-dns link vnet create -g ${RESOURCE_GROUP} -z ${CLUSTER_NAME}.${BASE_DOMAIN} -n ${INFRA_ID}-network-link -v "${INFRA_ID}-vnet" -e false`
    ```
    export VHD_BLOB_URL=`az storage blob url --account-name ${STORAGESA} --account-key ${ACCOUNT_KEY} -c vhd -n "rhcos.vhd" -o tsv`
    ```

14. Setup access to STORAGE BLOB \
`az deployment group create -g ${RESOURCE_GROUP} --template-file "02_storage.json" --parameters vhdBlobURL="${VHD_BLOB_URL}" --parameters baseName="${INFRA_ID}"`

15. Setup INFRA LOAD-BALANCERS & DNS \
`az deployment group create -g ${RESOURCE_GROUP}  --template-file "03_infra.json"  --parameters privateDNSZoneName="${CLUSTER_NAME}.${BASE_DOMAIN}"  --parameters baseName="${INFRA_ID}"`

    ```
    export PUBLIC_IP=`az network public-ip list -g ${RESOURCE_GROUP} --query "[?name=='${INFRA_ID}-master-pip'] | [0].ipAddress" -o tsv`
    ```
    `az network dns record-set a add-record -g ${BASE_DOMAIN_RESOURCE_GROUP} -z ${BASE_DOMAIN} -n api.${CLUSTER_NAME} -a ${PUBLIC_IP} --ttl 60`


16. Setup the BOOTSTRAP NODE \
    ```
    export BOOTSTRAP_URL=`az storage blob url --account-name ${STORAGESA} --account-key ${ACCOUNT_KEY} -c "files" -n "bootstrap.ign" -o tsv`

    export BOOTSTRAP_IGNITION=`jq -rcnM --arg v "2.2.0" --arg url ${BOOTSTRAP_URL} '{ignition:{version:$v,config:{replace:{source:$url}}}}' | base64 -w0`
    ```

    `az deployment group create -g ${RESOURCE_GROUP} --template-file "04_bootstrap.json" --parameters bootstrapIgnition="${BOOTSTRAP_IGNITION}" --parameters sshKeyData="${SSH_KEY}" --parameters baseName="${INFRA_ID}" `

17. Setup the MASTER NODES
    ```
    export MASTER_IGNITION=`cat redwagon/master.ign | base64 -w0`
    ```
    `az deployment group create -g ${RESOURCE_GROUP} --template-file "05_masters.json" --parameters masterIgnition="${MASTER_IGNITION}" --parameters sshKeyData="${SSH_KEY}" --parameters privateDNSZoneName="${CLUSTER_NAME}.${BASE_DOMAIN}" --parameters baseName="${INFRA_ID}"`

    Monitor progress : `openshift-install wait-for bootstrap-complete --dir=${INSTALL_DIR} --log-level info`


18. Setup the WORKER NODES
    ```
    export WORKER_IGNITION=`cat redwagon/worker.ign | base64 -w0`
    ```
    `az deployment group create -g ${RESOURCE_GROUP} --template-file "06_workers.json" --parameters workerIgnition="${WORKER_IGNITION}" --parameters sshKeyData="${SSH_KEY}" --parameters baseName="${INFRA_ID}" `

19. Approve WORKER certificate requests, twice. \
Each worker node will submit a request marked 'Pending' two times, each certificate must be approved.

    `oc get csr | grep Pending` \
    `oc adm certificate approve csr-XXXXX csr-XXXXX csr-XXXXX`
