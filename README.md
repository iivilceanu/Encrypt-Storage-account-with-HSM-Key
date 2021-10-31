# Encrypt-Storage-account-with-HSM-Key

#Change the execution policy to unblock importing AzFilesHybrid.psm1 module
Set-ExecutionPolicy -ExecutionPolicy Unrestricted -Scope CurrentUser

#Powershell version
$PSVersionTable.PSVersion

#Import module AZ - if not installed
Install-Module -Name Az 

#Install module Keyvault - if not installed 
Install-Module -Name Az.Keyvault
 
#Check if the module AZ is installed
Get-InstalledModule -Name Az*

#Login to Azure 
  az login

#Set Azure Subscription
az account set --subscription subscriptionid

#Create a resource group
az group create --name "resourcename" --location selectedlocation

#Create an HSM in your location with the proper purge policy to avoid the error "az : ERROR: BadRequestError: (KeyVaultPolicyError) Keyvault policy recoverable is not set"  
 oid=$(az ad signed-in-user show --query objectId -o tsv)
 az keyvault create --hsm-name "iliehsm4" --resource-group "Resour1" --location "West Europe" --administrators $oid --retention-days 28 --enable-purge-protection true --enabled-for-deployment true --enabled-for-template-deployment true

#Activate your HSM - https://docs.microsoft.com/en-us/azure/key-vault/managed-hsm/quick-create-cli#activate-your-managed- https://docs.microsoft.com/en-us/azure/key-vault/managed-hsm/quick-create-cli 
openssl req -newkey rsa:2048 -nodes -keyout cert_0.key -x509 -days 365 -out cert_0.cer
openssl req -newkey rsa:2048 -nodes -keyout cert_1.key -x509 -days 365 -out cert_1.cer
openssl req -newkey rsa:2048 -nodes -keyout cert_2.key -x509 -days 365 -out cert_2.cer
az keyvault security-domain download --hsm-name hsmname --sd-wrapping-keys /home/ilie/cert_0.cer /home/ilie/cert_1.cer /home/ilie/cert_2.cer --sd-quorum 2 --security-domain-file hsmname.json

#assign you user the Crypto User Role to be able to create the key:
#![image](https://user-images.githubusercontent.com/71116646/139575288-16fe624a-73ed-4c92-8b7b-8d6852b32efd.png)

 az ad user show --id userid@xxx.com --query "objectId"
   az keyvault role assignment create --hsm-name hsmname --role "515eb02d-2335-4d2d-92f2-b1cbdf9c3778" --scope / --assignee-object-id "objectid"
   az keyvault role assignment create --hsm-name hsmname --role "21dbd100-6940-42c2-9190-5d6cb909625b" --scope / --assignee-object-id "objectid"
#Alternatively assign your user by name
az keyvault role assignment create --hsm-name hsmname --role "Managed HSM Crypto Service Encryption User" --assignee iliev@microsoft.com  --scope /keys

#Create a key https://docs.microsoft.com/en-us/azure/key-vault/managed-hsm/key-management#create-an-hsm-key
 az keyvault key create --hsm-name hsmname --name myrsakey --ops encrypt wrapKey unwrapKey --kty RSA-HSM --size 307

#assign the identity to the storage account https://github.com/MicrosoftDocs/azure-docs/blob/master/articles/storage/common/customer-managed-keys-configure-key-vault-hsm.md
az storage account update --name storageaccountname --resource-group Resour1 --assign-identity

#Assign a role to the storage account for access to the managed HSM https://github.com/MicrosoftDocs/azure-docs/blob/master/articles/storage/common/customer-managed-keys-configure-key-vault-hsm.md
$storage_account_principal = $(az storage account show --name storageaccountanme --resource-group resourcegroupnameofstorageaccount --query identity.principalId --output tsv)

#Assign a role to the storage account for access to the managed HSM https://github.com/MicrosoftDocs/azure-docs/blob/master/articles/storage/common/customer-managed-keys-configure-key-vault-hsm.md#assign-a-role-to-the-storage-account-for-access-to-the-managed-hsm
az keyvault role assignment create --hsm-name hsmname --role "Managed HSM Crypto Service Encryption User" --assignee $storage_account_principal --scope /keys

#Configure encryption with a key in the managed HSM https://github.com/MicrosoftDocs/azure-docs/blob/master/articles/storage/common/customer-managed-keys-configure-key-vault-hsm.md#configure-encryption-with-a-key-in-the-managed-hsm
$hsmurl = $(az keyvault show --hsm-name hsmname--query properties.hsmUri --output tsv)
az storage account update --name storageaccountanme --resource-group resourcegroupnameofstorageaccount --encryption-key-name myrsakey --encryption-key-source Microsoft.Keyvault --encryption-key-vault $hsmurl
