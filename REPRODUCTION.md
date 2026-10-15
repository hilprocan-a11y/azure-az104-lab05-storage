# Reproduce Lab05 with Bicep

The template was deployed and verified on October 14, 2026 (Edmonton), then cleaned up. Commands below target **Windows CMD**, not PowerShell. See EVIDENCE.md for observed results.

## Prerequisites and scope

- Azure CLI with Bicep support, signed in to the intended subscription.
- Permissions to create storage resources and assign the Blob data role.
- A globally available storage account name and the client's current public IPv4 address.

The example subscription name is Azure for Students; replace it if yours differs. Change to the repository's bicep directory so main.bicep resolves correctly. The account name defaults to sthnlab05storage; if unavailable, select another name and consistently update subsequent commands.

The template provisions infrastructure and configuration, not historical data, versions, snapshots, or SAS tokens. Shared Key access is retained for the Azure Files exercises; no identity-based Files configuration is deployed.

## 1. Check inputs and compile

~~~cmd
az account show --query "{Subscription:name,State:state}" --output table
az ad signed-in-user show --query id --output tsv
curl.exe -4 https://api.ipify.org
az storage account check-name --name sthnlab05storage
az bicep build --file main.bicep
~~~

Set these variables to verified values. Replace the markers before running:

~~~cmd
set "LAB05_IPV4=YOUR_CURRENT_PUBLIC_IPV4"
set "LAB05_USER_ID=YOUR_ENTRA_USER_OBJECT_ID"
~~~

These variables apply only to the current CMD session. The user Object ID is not a subscription ID or a built-in role ID.

## 2. Create the resource group

~~~cmd
az group create --subscription "Azure for Students" --name rg-hn-az104-lab05-storage --location centralus --tags Environment=Training Lab=Lab05 Project=AZ104 Owner=Hilaire
~~~

The template targets resourceGroup; it does not create this group itself. Resource tags are also declared in Bicep because group tags are not automatically inherited.

## 3. Validate and preview

~~~cmd
az deployment group validate --subscription "Azure for Students" --resource-group rg-hn-az104-lab05-storage --template-file main.bicep --parameters clientIpv4=%LAB05_IPV4% blobUserObjectId=%LAB05_USER_ID%
~~~

~~~cmd
az deployment group what-if --subscription "Azure for Students" --resource-group rg-hn-az104-lab05-storage --template-file main.bicep --parameters clientIpv4=%LAB05_IPV4% blobUserObjectId=%LAB05_USER_ID%
~~~

Review the actual preview. An empty lab group should show seven creations. Neither command applies the resources described in the template.

## 4. Deploy

~~~cmd
az deployment group create --subscription "Azure for Students" --resource-group rg-hn-az104-lab05-storage --name lab05-storage-bicep --template-file main.bicep --parameters clientIpv4=%LAB05_IPV4% blobUserObjectId=%LAB05_USER_ID%
~~~

Wait for provisioningState: Succeeded. If choosing another account name, add storageAccountName=YOUR_CHOSEN_NAME to the parameters in validate, what-if, and create.

## 5. Read back the deployed configuration

~~~cmd
az storage account show --subscription "Azure for Students" --resource-group rg-hn-az104-lab05-storage --name sthnlab05storage --query "{State:provisioningState,SKU:sku.name,Tier:accessTier,HTTPS:enableHttpsTrafficOnly,TLS:minimumTlsVersion,Anonymous:allowBlobPublicAccess,PublicNetwork:publicNetworkAccess,Rules:networkRuleSet}" --output json
~~~

~~~cmd
az storage account blob-service-properties show --subscription "Azure for Students" --resource-group rg-hn-az104-lab05-storage --account-name sthnlab05storage --query "{Versioning:isVersioningEnabled,BlobSoftDelete:deleteRetentionPolicy,ContainerSoftDelete:containerDeleteRetentionPolicy}" --output json
~~~

~~~cmd
az storage account file-service-properties show --subscription "Azure for Students" --resource-group rg-hn-az104-lab05-storage --account-name sthnlab05storage --query shareDeleteRetentionPolicy --output json
~~~

~~~cmd
az storage share-rm show --subscription "Azure for Students" --resource-group rg-hn-az104-lab05-storage --storage-account sthnlab05storage --name partage-lab05 --query "{Name:name,Protocol:enabledProtocols,Tier:accessTier,QuotaGiB:shareQuota}" --output json
~~~

~~~cmd
az storage account management-policy show --subscription "Azure for Students" --resource-group rg-hn-az104-lab05-storage --account-name sthnlab05storage --query policy.rules --output json
~~~

Expect LRS/Hot, HTTPS/TLS1.2, private Blob access, default-deny firewall with the supplied IP, versioning and seven-day retention policies, a 100 GiB SMB share, and the 30-day lifecycle rule. Reading the policy confirms configuration, not automatic execution.

## 6. Verify Blob access and content

Use new local filenames if they already exist. The first command creates or replaces the named local test file.

~~~cmd
echo AZ-104 Lab05 - Bicep deployment verified>"%USERPROFILE%\lab05-bicep-test.txt"
~~~

~~~cmd
az storage blob upload --account-name sthnlab05storage --container-name documents --name lab05-bicep-test.txt --file "%USERPROFILE%\lab05-bicep-test.txt" --auth-mode login
~~~

~~~cmd
az storage blob download --account-name sthnlab05storage --container-name documents --name lab05-bicep-test.txt --file "%USERPROFILE%\lab05-bicep-downloaded.txt" --auth-mode login
~~~

~~~cmd
type "%USERPROFILE%\lab05-bicep-downloaded.txt"
~~~

Expected text: AZ-104 Lab05 - Bicep deployment verified. The tested CMD file was 42 bytes. RBAC propagation may take time; distinguish authorization failures from network-rule failures before changing permissions. These commands exercise Blob access, not Files access.

Run the same what-if command again after deployment. Review differences rather than assuming the preview must be empty; the observed default-property differences are documented in EVIDENCE.md.

## 7. Clean up

Save the required evidence first. Do not publish credentials or complete SAS URLs. Inspect the group before deleting it:

~~~cmd
az resource list --subscription "Azure for Students" --resource-group rg-hn-az104-lab05-storage --query "[].{Name:name,Type:type}" --output table
~~~

The following command deletes the whole lab group and everything inside it. Confirm only after checking its contents:

~~~cmd
az group delete --subscription "Azure for Students" --name rg-hn-az104-lab05-storage
~~~

After the command finishes:

~~~cmd
az group exists --subscription "Azure for Students" --name rg-hn-az104-lab05-storage
~~~

Expect false. Local source and evidence files are unaffected. Previously incurred charges are not cancelled by deletion.
