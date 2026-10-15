# Verification summary

October 14, 2026 (Edmonton). This is an English summary of user-provided results, not a verbatim transcript. Original evidence is retained separately in the local archive.

| Bicep verification | Observed result |
|---|---|
| Compilation | Exit code 0, no diagnostics |
| Azure validation | Succeeded, error null |
| Initial what-if | 7 creations |
| Deployment | Succeeded, 7 returned resources, approximately 25 seconds |
| Account | StorageV2, Standard_LRS, Hot, Central US |
| Transport and anonymous access | HTTPS true, TLS1_2, allowBlobPublicAccess false |
| Firewall | Public endpoint Enabled, default Deny, bypass None, one allowed client IPv4 |
| Blob protections | Versioning true; blob/container soft delete enabled for 7 days |
| Azure Files | Share soft delete 7 days; SMB, TransactionOptimized, 100 GiB |
| Lifecycle | documents/, blockBlob, current blobs to Cool after more than 30 days since modification |
| Blob data access | Entra upload/download succeeded; 42 bytes, matching MD5 |
| Downloaded content | AZ-104 Lab05 - Bicep deployment verified |
| Final what-if | 5 no change, 2 to modify on omitted defaults; no second deployment applied |
| Cleanup | az group exists returned false |

## Final what-if differences

The preview compared Azure-returned values against properties omitted from the template:

- Blob service: deleteRetentionPolicy.allowPermanentDelete: false, staticWebsite.enabled: false.
- Container: defaultEncryptionScope: $account-encryption-key, denyEncryptionScopeOverride: false.
- Noeffect was reported for encryption.services and principalType.

No resource deletion was predicted. The result is not presented as a zero-difference preview, and the effect of a future update on omitted properties is not assumed from this preview alone.

## Interpretation limits

The deployment confirms creation of the Blob Data Contributor assignment. Successful transfers do not isolate this assignment from inherited permissions. Files transfers and automatic lifecycle execution were not tested during the Bicep reconstruction. Earlier Files snapshot recovery and portal/CLI data protection exercises are described separately in README.md.

## Cleanup confirmation

The pre-deletion inventory contained only the lab storage account. The user confirmed resource-group deletion, then checked its existence. The result was false. Local source files and original evidence were retained.
