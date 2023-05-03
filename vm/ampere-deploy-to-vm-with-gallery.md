# Create and Customize an ARM64 Azure VM from an Azure Compute Gallery

This guide will walk you through the process of creating a virtual machine (VM) and a new image from an Azure Compute Image Gallery based on an existing VM. The process is divided into four steps:

1. Set up the Environment
2. Create a VNET, subnet, and VM
3. Create a new customized VM image in an Azure compute gallery, based on the VM you just created
4. Create a VM from the image gallery

## Prerequisites

- An Azure account with a subscription ID. Sign up for a free account [here](https://azure.microsoft.com/en-us/free/).
- Install the Azure CLI. Follow the instructions [here](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest).

## Step 1: Set up the Environment

Export the following environment variables, replacing `<your ...>` with your actual values:

```bash
export RESOURCE_GROUP_NAME=<your resource group name>
export LOCATION=<your location>
export NETWORK_SUBNET_NAME=<your subnet name>
export NETWORK_NAME=<your vnet name>
export NETWORK_SECURITY_GROUP=<your NSG name>
export VM_NAME=<your VM name>
export SOURCE_VM_NAME=<your source VM name>
export SOURCE_RESOURCE_GROUP_NAME=<your source resource group name>
export TARGET_VM_NAME=<your target VM name>
export VM_IMAGE=<your VM image name>
export VM_IMAGE_VERSION=<your VM image version>
export IMAGE_GALLERY_NAME=<your image gallery name>
```

Set your subscription ID:

```bash
export SUBSCRIPTION_ID=<your subscription ID: az account image show>
```

Set the ARM64 image to use for the VM:

```bash
export sourcearmimage=<your ARM image name>
export sourcearmimagename=<your ARM image name>
```

## Step 2: Create a VNET, Subnet, and VM

1. Create a resource group:

   ```bash
   az group create --resource-group $RESOURCE_GROUP_NAME --location $LOCATION
   ```

2. Create a VNET:

   ```bash
   az network vnet create --resource-group $RESOURCE_GROUP_NAME --location $LOCATION --name $NETWORK_NAME --address-prefixes 172.0.0.0/16
   ```

3. Create a subnet:

   ```bash
   az network vnet subnet create --resource-group $RESOURCE_GROUP_NAME --vnet-name $NETWORK_NAME --address-prefixes 172.0.0.0/24 --name $NETWORK_SUBNET_NAME
   ```

4. Create a VM:

   ```bash
   az vm create \
      --resource-group $RESOURCE_GROUP_NAME \
      --location $LOCATION \
      --name $SOURCE_VM_NAME \
      --image $sourcearmimagename \
      --vnet-name $NETWORK_NAME \
      --subnet $NETWORK_NAME \
      --public-ip-sku Standard \
      --generate-ssh-keys \
      --specialized
   ```

5. Install Cassandra on the new VM via the serial console:

   ```bash
   # Replace 'ubuntu_release' with the actual Ubuntu release number
   Ubuntu_release=`lsb_release -rs`
   wget https://packages.microsoft.com/config/ubuntu/${ubuntu_release}/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
   sudo dpkg -i packages-microsoft-prod.deb

   sudo apt-get install apt-transport-https
   sudo apt-get update
   sudo apt-get install msopenjdk-8

   echo "deb http://www.apache.org/dist/cassandra/debian 311x main" | sudo tee -a /etc/apt/sources.list.d/cassandra.sources.list
   curl https://www.apache.org/dist/cassandra/KEYS | sudo apt-key add -
   sudo apt update
   sudo apt install cassandra
   sudo service cassandra start
   sudo service cassandra status
   cqlsh
   ```

## Step 3: Create a New Customized VM Image in an Azure Compute Gallery, Based on the VM You Just Created

1. Create an Azure Compute image gallery:

   ```bash
   az sig create --resource-group $RESOURCE_GROUP_NAME --gallery-name $IMAGE_GALLERY_NAME
   ```

2. Create the Image Definition and add it to the gallery:

   ```bash
   az sig image-definition create \
      --resource-group $RESOURCE_GROUP_NAME \
      --location $LOCATION \
      --gallery-name $IMAGE_GALLERY_NAME \
      --gallery-image-definition $VM_IMAGE \
      --publisher Canonical \
      --offer UbuntuServer \
      --sku ${sourcearmimagename} \
      --os-type Linux \
      --os-state specialized \
      --hyper-v-generation V2 \
      --architecture Arm64
   ```

3. Get the ID of the source VM to use as an image:

   ```bash
   az vm get-instance-view -g $RESOURCE_GROUP_NAME -n $VM_NAME --query id
   ```

4. Create the Image Version:

   ```bash
   az sig image-version create \
      --resource-group $RESOURCE_GROUP_NAME \
      --location $LOCATION \
      --gallery-name $IMAGE_GALLERY_NAME \
      --gallery-image-definition $VM_IMAGE \
      --gallery-image-version $VM_IMAGE_VERSION \
      --target-regions "East US" \
      --replica-count 1 \
      --virtual-machine "/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${SOURCE_RESOURCE_GROUP_NAME}/providers/Microsoft.Compute/virtualMachines/${SOURCE_VM_NAME}"
   ```

## Step 4: Create a VM from the Image Gallery

1. Create a VM:

   ```bash
   az vm create \
      --resource-group $RESOURCE_GROUP_NAME \
      --location $LOCATION \
      --name $TARGET_VM_NAME \
      --image "/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${RESOURCE_GROUP_NAME}/providers/Microsoft.Compute/galleries/${IMAGE_GALLERY_NAME}/images/${IMAGE_GALLERY_NAME}" \
      --vnet-name $NETWORK_NAME \
      --subnet $NETWORK_NAME \
      --public-ip-sku Standard \
      --generate-ssh-keys \
      --specialized
   ```

2. To view the created VM in the Azure portal, navigate to:

   ```
   https://portal.azure.com/?feature.customportal=false#@microsoft.onmicrosoft.com/resource/subscriptions/c5bd7244-ab30-4473-8e88-d4d58b246c94/resourceGroups/armvmdemo/providers/Microsoft.Compute/virtualMachines/armvmfromgallery/overview
   ```

3. Connect to the VM via the serial console to see the running app.

### Optional: Open Ports for HTTP Access

```bash
az vm open-port --resource-group $RESOURCE_GROUP_NAME \
   --location $LOCATION \
   --name $TARGET_VM_NAME \
   --port 2222,8080 --priority 100
```

That's it! You've successfully created a VM, a new customized VM image in an Azure Compute Gallery based on the existing VM, and a new VM from the image gallery.

You may want to explore additional customization options based on your specific requirements. Some examples include:

- Installing additional software packages.
- Configuring system settings.
- Setting up database or application servers.
- Connecting the VM to other Azure services.

Remember to monitor and maintain the VM by regularly checking its performance, security, and updating the installed software.