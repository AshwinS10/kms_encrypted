## 0.3.0

- Added support for Vault
- Removed `KmsEncrypted.kms_client` and `KmsEncrypted.client_options` in favor of `KmsEncrypted.aws_client`
- Removed `KmsEncrypted::Google.kms_client` in favor of `KmsEncrypted.google_client`

## 0.2.0

- Added support for Google KMS

## 0.1.4

- Added `kms_keys` method to models
- Reset data keys when record is reloaded
- Added `kms_client`
- Added ActiveSupport notifications

## 0.1.3

- Added test key
- Added `client_options`
- Allow private or protected `kms_encryption_context` method

## 0.1.2

- Use `KMS_KEY_ID` env variable by default

## 0.1.1

- Added key rotation
- Added support for multiple keys per record

## 0.1.0

- First release
