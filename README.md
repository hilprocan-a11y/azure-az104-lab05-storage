# Lab05 — Azure Storage and Data Protection

Hilaire's personal AZ-104 learning path. Portal practice: September 14–15, 2026. CLI practice: October 10. Bicep reconstruction: October 14 (Edmonton dates). Lab numbering belongs to this personal path, not the official Microsoft lab catalog.

**Status: Bicep deployed and verified; resource group deleted after testing.**

## Objective

Configure Azure Storage, authorize data access, recover files after changes or deletion, and reproduce the infrastructure with Bicep. Distinguish infrastructure redundancy from data recovery, and identity permissions from network access.

## Architecture

~~~mermaid
flowchart TD
  USER[Client with Microsoft Entra identity] --> FW[Public endpoint / firewall allows client IPv4]
  FW --> SA[StorageV2 / Standard_LRS / Hot]
  SA --> BLOB[Private Blob container: documents]
  SA --> FILES[SMB share: partage-lab05 / 100 GiB]
  BLOB --> BP[Versioning / blob and container soft delete: 7 days]
  FILES --> FP[Share soft delete: 7 days]
  SA --> LC[Lifecycle: current blobs under documents/ to Cool after 30 days]
  RBAC[Storage Blob Data Contributor / account scope] -. authorizes Blob access .-> BLOB
~~~

The account requires HTTPS and TLS 1.2, disables anonymous Blob access, and uses Microsoft-managed encryption keys. The public endpoint is enabled with a default-deny firewall and the current client IPv4 allowed. Shared Key access remains enabled for the Azure Files exercises.

The CLI/Bicep share has a **100 GiB quota**. The earlier portal exercise displayed a 100 TiB maximum capacity; this is not the same setting or reconstruction configuration.

## Verified exercises

| Exercise | Observed result | Practice stage |
|---|---|---|
| Blob access with Microsoft Entra ID | Upload/download succeeded; local content checked | Portal, CLI, Bicep reconstruction |
| Restore previous content | Recovered Version 1 after modifying the document | Portal, CLI |
| Delete the current blob | Preserved versions allowed a current blob to be recreated | Portal, CLI |
| Soft-delete a previous version | Deleted version restored with Undelete; temporary Blob Data Owner role removed afterward | Portal |
| User delegation SAS | Read succeeded, anonymous access denied, same URL denied after expiry | Portal, CLI |
| Storage firewall | Allowed IP succeeded; removing it blocked access; adding it restored access | Portal, CLI |
| Azure Files recovery | Snapshot created, live file deleted, original content recovered and returned to the share | Portal, CLI |
| Access tiers | Original blob changed from inherited Hot to explicit Cool | CLI; inspected in Storage Explorer |
| AzCopy | Entra-authenticated download and new upload succeeded; 71 bytes | CLI |
| Storage Explorer | Browsed blobs, previewed content, downloaded Version 1 | Windows application |
| Lifecycle policy | Configuration saved and read back; automatic execution not observed | CLI, Bicep |

Azure Files data operations used an account key. They do not demonstrate identity-based Files access or an SMB mount from Windows. The Blob role does not grant Files permissions.

## Bicep reconstruction

[main.bicep](bicep/main.bicep) declares seven resources: storage account, Blob service, private container, File service, SMB share, lifecycle policy, and RBAC assignment. The resource group is created separately through Azure CLI.

| Parameter | Purpose | How to obtain it |
|---|---|---|
| storageAccountName | Globally unique account name | Choose a name and run az storage account check-name |
| location | Azure region | Defaults to the resource group's region |
| clientIpv4 | Allowed public client address | curl.exe -4 https://api.ipify.org |
| blobUserObjectId | Microsoft Entra user receiving the Blob role | az ad signed-in-user show --query id --output tsv |
| tags | Training resource classification | Defaults provided in the template |

The parent property expresses resource hierarchy. The role assignment's scope limits it to the account. The guid function generates a stable assignment name from the account, user, and built-in role identifiers.

The template recreates **configuration**, not the old files, versions, snapshots, or SAS tokens deleted with the previous account. See [REPRODUCTION.md](REPRODUCTION.md) for CMD commands, checks, and cleanup.

### October 14 deployment results

1. Local compilation: no errors.
2. Azure validation: Succeeded, no error.
3. Initial what-if: seven expected creations.
4. Deployment lab05-storage-bicep: Succeeded, seven resources, approximately 25 seconds.
5. Account, firewall, Blob protections, Files share, and lifecycle properties matched the template.
6. Entra upload/download: 42 bytes, matching MD5, verified text: AZ-104 Lab05 - Bicep deployment verified.
7. Final what-if: five resources unchanged; two differences reviewed.
8. Resource group deleted; az group exists returned false.

Final what-if differences concerned Azure-returned defaults omitted from the template: allowPermanentDelete, staticWebsite, defaultEncryptionScope, and denyEncryptionScopeOverride. No resource deletion was predicted. This was **not a strictly zero-difference preview**, and no second deployment was applied based on those differences.

Successful transfers demonstrate effective Blob access. They do not isolate the newly created assignment from inherited permissions. The deployment output separately confirms creation of the intended RBAC assignment. See [EVIDENCE.md](EVIDENCE.md) for the verification summary.

The English Bicep comments and parameter descriptions were translated after deployment. Resource declarations and parameter values were not changed by translation.

## Troubleshooting and lessons learned

- **Empty first upload:** a zero-byte version was identified. The local file was corrected, uploaded again, and downloaded for content verification. Command success alone does not prove correct content.
- **Entra denial on Azure Files:** the Blob role did not authorize Files access. Switching to Access key allowed the intended portal exercise.
- **Expired SAS:** read succeeded before expiry and failed afterward. A message comparing start and expiry times does not, by itself, prove that a token was regenerated.
- **Wrong local copy:** an existing Version 2 file was opened after a download conflict. Azure Preview and a download to an empty folder confirmed Version 1.
- **Missing table column:** blobTierInferred was null in one CLI response after a tier change. A missing column must not be interpreted as false.
- **Redundancy is not recovery:** LRS also replicates deletions. Versions, snapshots, and soft delete address accidental data loss.

## Scope and remaining topics

Container and share soft delete were configured, but deleting and recovering an entire container or share was not tested. Files recovery restored an individual file from a share snapshot.

Archive rehydration, geographic failover, object replication, customer-managed encryption keys, and identity-based Files access were not deployed. SAS types, stored access policies, key rotation, and redundancy choices remain targeted study topics. Private endpoints will be addressed in the advanced networking lab.

Guided quizzes were followed by reasoning scenarios; initial confusion about LRS and GRS/GZRS was corrected. These exercises are not a mock-exam score or evidence of exhaustive AZ-104 storage coverage.

## Evidence and cleanup

Original portal screenshots, CLI outputs, and Bicep deployment records are retained in a separate local archive. Raw originals are not published here because some contain account identifiers or SAS URL parameters. The English evidence summary describes observed results without those values.

Each practice phase was cleaned up. The October 14 Bicep resource group was deleted, with false confirming absence. Local files and the template were retained. This does not claim a subscription-wide resource or credit audit.

## Microsoft references

- [Storage account Bicep reference](https://learn.microsoft.com/en-us/azure/templates/microsoft.storage/2025-06-01/storageaccounts)
- [Lifecycle policy Bicep reference](https://learn.microsoft.com/en-us/azure/templates/microsoft.storage/2025-06-01/storageaccounts/managementpolicies)
- [Blob access tiers](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview)
- [AzCopy](https://learn.microsoft.com/en-us/azure/storage/common/storage-use-azcopy-v10)
- [Storage Explorer](https://azure.microsoft.com/en-us/products/storage/storage-explorer/)
