# AKS Private Link Testing

A test repo for working with multi-cluster AKS communicating over private link service and endpoints.

## Setup

```bash

# Setup some variables to use
LOCATION=centralus

PREFIX=cdw
SUFFIX=20220709

SOURCE_NAME=$PREFIX-cluster1-$SUFFIX 
TARGET_NAME=$PREFIX-cluster2-$SUFFIX 

# Repeat for both SOURCE_NAME and TARGET_NAME
BASE_NAME=$PREFIX-$VALUE1-$SUFFIX

# Create the resource group
az group create -l $LOCATION -n $BASE_NAME

# Create the cluster
az aks create \
-g $BASE_NAME \
-n $BASE_NAME \
--generate-ssh-keys \
--enable-managed-identity \
--node-count 1 \
--network-plugin azure \
--enable-addons monitoring

```

## Deploy App in Cluster #1

Notice how the deployment.yaml file has service annotations that create the Private Link Service for us.

```bash

az aks get-credentials -n $SOURCE_NAME -g $SOURCE_NAME

kubectl apply -f deployment.yaml

```

## Create a Private Endpoint in VNET #2

```bash

SOURCE_AKS_MC_RG=$(az aks show -g $SOURCE_NAME --name $SOURCE_NAME --query nodeResourceGroup -o tsv)
SOURCE_AKS_PLS_ID=$(az network private-link-service list -g ${SOURCE_AKS_MC_RG} --query "[].id" -o tsv)

TARGET_AKS_MC_RG=$(az aks show -g $TARGET_NAME --name $TARGET_NAME --query nodeResourceGroup -o tsv)
TARGET_AKS_MC_VNET=$(az network vnet list -g $TARGET_AKS_MC_RG | jq -r '.[0].name')
TARGET_AKS_MC_SUBNET=$(az network vnet subnet list -g $TARGET_AKS_MC_RG --vnet-name $TARGET_AKS_MC_VNET | jq -r '.[0].name')

az network private-endpoint create \
    --name $TARGET_NAME-pe \
    --resource-group $TARGET_AKS_MC_RG \
    --connection-name PLEtoSourceCluster \
    --private-connection-resource-id $SOURCE_AKS_PLS_ID \
    --vnet-name $TARGET_AKS_MC_VNET \
    --subnet $TARGET_AKS_MC_SUBNET 

```

## Test Out the Results

```bash

az aks get-credentials -n $TARGET_NAME -g $TARGET_NAME

kubectl run -i --tty busybox --image=radial/busyboxplus:curl --restart=Never

curl http://10.224.0.35:9898/ 

```
