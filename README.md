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
az account set --subscription 17e9205a-8948-4a20-bb06-e19bf176af49

#Create a resource group
az group create --name "Reour2" --location westwuerope

#Create an HSM in your location with the proper purge policy to avoid the error "az : ERROR: BadRequestError: (KeyVaultPolicyError) Keyvault policy recoverable is not set"  
 oid=$(az ad signed-in-user show --query objectId -o tsv)
 az keyvault create --hsm-name "iliehsm4" --resource-group "Resour1" --location "West Europe" --administrators $oid --retention-days 28 --enable-purge-protection true --enabled-for-deployment true --enabled-for-template-deployment true

#Activate your HSM - https://docs.microsoft.com/en-us/azure/key-vault/managed-hsm/quick-create-cli#activate-your-managed- https://docs.microsoft.com/en-us/azure/key-vault/managed-hsm/quick-create-cli 
openssl req -newkey rsa:2048 -nodes -keyout cert_0.key -x509 -days 365 -out cert_0.cer
openssl req -newkey rsa:2048 -nodes -keyout cert_1.key -x509 -days 365 -out cert_1.cer
openssl req -newkey rsa:2048 -nodes -keyout cert_2.key -x509 -days 365 -out cert_2.cer
az keyvault security-domain download --hsm-name iliehsm4 --sd-wrapping-keys /home/ilie/cert_0.cer /home/ilie/cert_1.cer /home/ilie/cert_2.cer --sd-quorum 2 --security-domain-file iliehsm4.json

#assign you user the Crypto User Role to be able to create the key:
 az ad user show --id iliev@microsoft.com --query "objectId"
   az keyvault role assignment create --hsm-name iliehsm --role "515eb02d-2335-4d2d-92f2-b1cbdf9c3778" --scope / --assignee-object-id "44459b2c-8d38-4b48-8c1b-3da61460efc3"
   az keyvault role assignment create --hsm-name iliehsm --role "21dbd100-6940-42c2-9190-5d6cb909625b" --scope / --assignee-object-id "44459b2c-8d38-4b48-8c1b-3da61460efc3"
#Alternatively assign your user by name
az keyvault role assignment create --hsm-name iliehsm --role "Managed HSM Crypto Service Encryption User" --assignee iliev@microsoft.com  --scope /keys

#Create a key https://docs.microsoft.com/en-us/azure/key-vault/managed-hsm/key-management#create-an-hsm-key
 az keyvault key create --hsm-name iliehsm12 --name myrsakey --ops encrypt wrapKey unwrapKey --kty RSA-HSM --size 307

#assign the identity to the storage account https://github.com/MicrosoftDocs/azure-docs/blob/master/articles/storage/common/customer-managed-keys-configure-key-vault-hsm.md
az storage account update --name fcsb --resource-group Resour1 --assign-identity

#Assign a role to the storage account for access to the managed HSM https://github.com/MicrosoftDocs/azure-docs/blob/master/articles/storage/common/customer-managed-keys-configure-key-vault-hsm.md
$storage_account_principal = $(az storage account show --name fcsb --resource-group Resour1 --query identity.principalId --output tsv)

#Assign a role to the storage account for access to the managed HSM https://github.com/MicrosoftDocs/azure-docs/blob/master/articles/storage/common/customer-managed-keys-configure-key-vault-hsm.md#assign-a-role-to-the-storage-account-for-access-to-the-managed-hsm
az keyvault role assignment create --hsm-name iliehsm4 --role "Managed HSM Crypto Service Encryption User" --assignee $storage_account_principal --scope /keys

#Configure encryption with a key in the managed HSM https://github.com/MicrosoftDocs/azure-docs/blob/master/articles/storage/common/customer-managed-keys-configure-key-vault-hsm.md#configure-encryption-with-a-key-in-the-managed-hsm
$hsmurl = $(az keyvault show --hsm-name iliehsm4 --query properties.hsmUri --output tsv)
az storage account update --name fcsb --resource-group Resour1 --encryption-key-name myrsakey --encryption-key-source Microsoft.Keyvault --encryption-key-vault $hsmurl
