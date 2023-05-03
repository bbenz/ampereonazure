# Create and Customize an ARM64 Azure VM with the Azure CLI

This guide will walk you through creating and customizing an ARM64 Azure VM using Azure CLI.

## Prerequisites

- An Azure account with a subscription ID. Sign up for a free account [here](https://azure.microsoft.com/en-us/free/).
- Install the Azure CLI. Follow the instructions [here](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest).

## Steps

1. Set up the environment.
2. (Optional but highly recommended) Create a VNET and subnet.
3. Create a VM.
4. Customize the VM.

### Step 1: Set up the Environment

Replace the placeholders with your desired values and run the following commands:

```bash
export RESOURCE_GROUP_NAME=<your resource group name>
export LOCATION=<your location>
export NETWORK_NAME=<your vnet name>
export NETWORK_SUBNET_NAME=<your subnet name>
export VM_NAME=<your VM name>
```

Find ARM64 images and set the desired image:

```bash
az vm image list \
    --all \
    --architecture Arm64 \
    -o table

az vm image list \
    --all \
    --architecture Arm64 \
    --publisher Canonical \
    -o table

export sourcearmimage=<your ARM image name>
export sourcearmimagename=<your ARM image name>
```

### Step 2: Create a VNET, subnet, and VM

Create a resource group:

```bash
az group create --resource-group $RESOURCE_GROUP_NAME --location $LOCATION
```

Create a VNET (optional but highly recommended):

```bash
az network vnet create --resource-group $RESOURCE_GROUP_NAME --location $LOCATION --name $NETWORK_NAME --address-prefixes 172.0.0.0/16
```

Create a subnet (optional but highly recommended):

```bash
az network vnet subnet create --resource-group $RESOURCE_GROUP_NAME --vnet-name $NETWORK_NAME --address-prefixes 172.0.0.0/24 --name $NETWORK_SUBNET_NAME
```

### Step 3: Create a VM

Create the VM using the following command:

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

(Optional) Open ports for HTTP access:

```bash
az vm open-port --resource-group $RESOURCE_GROUP_NAME \
   --location $LOCATION \
   --name $TARGET_VM_NAME \
   --port 2222,8080 --priority 100
```

### Step 4: Customize the VM

In this example, we'll install Cassandra on the new VM via the serial console.

```bash
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

After completing these steps, you will have successfully created and customized an ARM64 Azure VM. You can now manage and use the VM for your desired purposes. 

You may want to explore additional customization options based on your specific requirements. Some examples include:

- Installing additional software packages.
- Configuring system settings.
- Setting up database or application servers.
- Connecting the VM to other Azure services.

Remember to monitor and maintain the VM by regularly checking its performance, security, and updating the installed software.