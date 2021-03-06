# Google KMS

Add this line to your application’s Gemfile:

```ruby
gem 'google-api-client'
gem 'kms_encrypted'
```

Add columns for the encrypted data and the encrypted KMS data keys

```ruby
add_column :users, :encrypted_email, :text
add_column :users, :encrypted_email_iv, :text
add_column :users, :encrypted_kms_key, :text
```

Create a [Google Cloud Platform](https://cloud.google.com/) account if you don’t have one. KMS works great whether or not you run your infrastructure on GCP.

Create a [KMS key ring and key](https://console.cloud.google.com/iam-admin/kms) and set it in your environment along with your GCP credentials ([dotenv](https://github.com/bkeepers/dotenv) is great for this)

```sh
KMS_KEY_ID=projects/.../locations/.../keyRings/.../cryptoKeys/...
```

And update your model

```ruby
class User < ApplicationRecord
  has_kms_key

  attr_encrypted :email, key: :kms_key
end
```

For each encrypted attribute, use the `kms_key` method for its key.

## Auditing

Follow the [instructions here](https://cloud.google.com/kms/docs/logging) to set up data access logging. To know what data is being decrypted, you’ll need to add context.

Add a `kms_encryption_context` method to your model.

```ruby
class User < ApplicationRecord
  def kms_encryption_context
    # some hash
  end
end
```

The context is used as part of the encryption and decryption process, so it must be a value that doesn’t change. Otherwise, you won’t be able to decrypt.

The primary key is a good choice, but auto-generated ids aren’t available until a record is created, and we need to encrypt before this. One solution is to preload the primary key. Here’s what it looks like with Postgres:

```ruby
class User < ApplicationRecord
  def kms_encryption_context
    self.id ||= self.class.connection.execute("select nextval('#{self.class.sequence_name}')").first["nextval"]
    {"Record" => "#{model_name}/#{id}"}
  end
end
```

## Alerting

We recommend setting up alerts on suspicious behavior.

## Key Rotation

To manually rotate keys, replace the old key id with the new key id in your model. Your app does not need the old key id to perform rotation (however, the key must still be enabled in your GCP account).

```sh
KMS_KEY_ID=...
```

and run

```ruby
User.find_each do |user|
  user.rotate_kms_key!
end
```

## Testing

For testing, you can prevent network calls to KMS by setting:

```sh
KMS_KEY_ID=insecure-test-key
```

## Multiple Keys Per Record

You may want to protect different columns with different data keys (or even master keys).

To do this, add more columns

```ruby
add_column :users, :encrypted_phone, :text
add_column :users, :encrypted_phone_iv, :text
add_column :users, :encrypted_kms_key_phone, :text
```

And update your model

```ruby
class User < ApplicationRecord
  has_kms_key
  has_kms_key name: :phone, key_id: "..."

  attr_encrypted :email, key: :kms_key
  attr_encrypted :phone, key: :kms_key_phone
end
```

For context, use:

```ruby
class User < ApplicationRecord
  def kms_encryption_context_phone
    # some hash
  end
end
```

To rotate keys, use:

```ruby
user.rotate_kms_key_phone!
```
