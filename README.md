# azure-upi-install
Azure UPI Install 

YouTube Demo : https://youtu.be/7-20Dq8eLS4
## Reference Links during the install
1. Red Hat OCP Tools (openshift-installer, oc, kubectl) and Pull Secret
    https://cloud.redhat.com
2. Azure CLI (az)
    https://docs.microsoft.com/en-us/cli/azure/install-azure-cli
3. JQ binary (also available via yum & apt in most distros)
    https://stedolan.github.io/jq/download/
4. Installation instructions used during the install, Azure w/ ARM templates.
    https://docs.openshift.com/container-platform/4.5/installing/installing_azure/installing-azure-user-infra.html


In this demonstration we setup our access to Azure by first loging in using the 'az login' command. Prereqs, would also required a Service Principal account be created, that can be found here (https://docs.openshift.com/container-platform/4.5/installing/installing_azure/installing-azure-user-infra.html#installation-azure-service-principal_installing-azure-user-infra). 

The goal of this install is to build a basic OCP 4.5 cluster in a pre-allocated resource group on Azure.

Please refer to [dude-script.md](.dude-script.md) for the procedures.
