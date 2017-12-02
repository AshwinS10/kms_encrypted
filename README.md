# KMS Encrypted

Simple, secure key management for [attr_encrypted](https://github.com/attr-encrypted/attr_encrypted)

The attr_encrypted gem is great for encryption, but:

1. Leaves you to manage the security of your keys
2. Doesn’t provide an easy way to rotate your keys
3. Doesn’t have a great audit trail to see how data has been accessed
4. Doesn’t let you grant encryption and decryption permission separately

[KMS](https://aws.amazon.com/kms/) addresses all of these issues and it’s easy to use them together.

[![Build Status](https://travis-ci.org/ankane/kms_encrypted.svg?branch=master)](https://travis-ci.org/ankane/kms_encrypted)

## How It Works

This approach uses KMS to manage encryption keys and attr_encrypted to do the encryption.

To encrypt an attribute, we first generate a [data key](http://docs.aws.amazon.com/kms/latest/developerguide/concepts.html) from our KMS master key. KMS sends both encrypted and unencrypted versions of the data key. We pass the unencrypted version to attr_encrypted and store the encrypted version in the `encrypted_kms_key` column. For each record, we generate a different data key.

To decrypt an attribute, we first decrypt the data key with KMS. Once we have the decrypted key, we pass it to attr_encrypted to decrypt the data. We can easily track decryptions since we have a different data key for each record.

## Getting Started

Add this line to your application’s Gemfile:

```ruby
gem 'kms_encrypted'
```

Add columns for the encrypted data and the encrypted KMS data keys

```ruby
add_column :users, :encrypted_email, :text
add_column :users, :encrypted_email_iv, :text
add_column :users, :encrypted_kms_key, :text
```

Create an [Amazon Web Services](https://aws.amazon.com/) account if you don’t have one. KMS works great whether or not you run your infrastructure on AWS.

Create a [KMS master key](https://console.aws.amazon.com/iam/home#/encryptionKeys) and set it in your environment ([dotenv](https://github.com/bkeepers/dotenv) is great for this)

```sh
KMS_KEY_ID=arn:aws:kms:...
```

You can also use the alias

```sh
KMS_KEY_ID=alias/my-alias
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

[AWS CloudTrail](https://aws.amazon.com/cloudtrail/) logs all decryption calls. However, to know what data is being decrypted, you’ll need to add context.

Add a `kms_encryption_context` method to your model.

```ruby
class User < ApplicationRecord
  def kms_encryption_context
    # some hash
  end
end
```

The context is used as part of the encryption and decryption process, so it must be a value that doesn’t change. Otherwise, you won’t be able to decrypt. Read more about [encryption context here](http://docs.aws.amazon.com/kms/latest/developerguide/encryption-context.html).

The primary key is a good choice, but auto-generated ids aren’t available until a record is created, and we need to encrypt before this. One solution is to preload the primary key. Here’s what it looks like with Postgres:

```ruby
class User < ApplicationRecord
  def kms_encryption_context
    self.id ||= self.class.connection.execute("select nextval('#{self.class.sequence_name}')").first["nextval"]
    {"Record" => "#{model_name}/#{id}"}
  end
end
```

[Amazon Athena](https://aws.amazon.com/athena/) is great for querying CloudTrail logs. Create a table (thanks to [this post](http://www.1strategy.com/blog/2017/07/25/auditing-aws-activity-with-cloudtrail-and-athena/) for the table structure) with:

```sql
CREATE EXTERNAL TABLE cloudtrail_logs (
    eventversion STRING,
    userIdentity STRUCT<
        type:STRING,
        principalid:STRING,
        arn:STRING,
        accountid:STRING,
        invokedby:STRING,
        accesskeyid:STRING,
        userName:String,
        sessioncontext:STRUCT<
            attributes:STRUCT<
                mfaauthenticated:STRING,
                creationdate:STRING>,
            sessionIssuer:STRUCT<
                type:STRING,
                principalId:STRING,
                arn:STRING,
                accountId:STRING,
                userName:STRING>>>,
    eventTime STRING,
    eventSource STRING,
    eventName STRING,
    awsRegion STRING,
    sourceIpAddress STRING,
    userAgent STRING,
    errorCode STRING,
    errorMessage STRING,
    requestId  STRING,
    eventId  STRING,
    resources ARRAY<STRUCT<
        ARN:STRING,
        accountId:STRING,
        type:STRING>>,
    eventType STRING,
    apiVersion  STRING,
    readOnly BOOLEAN,
    recipientAccountId STRING,
    sharedEventID STRING,
    vpcEndpointId STRING,
    requestParameters STRING,
    responseElements STRING,
    additionalEventData STRING,
    serviceEventDetails STRING
)
ROW FORMAT SERDE 'com.amazon.emr.hive.serde.CloudTrailSerde'
STORED  AS INPUTFORMAT 'com.amazon.emr.cloudtrail.CloudTrailInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION  's3://my-cloudtrail-logs/'
```

Change the last line to point to your CloudTrail log bucket and query away

```sql
SELECT
    eventTime,
    userIdentity.userName,
    requestParameters
FROM
    cloudtrail_logs
WHERE
    eventName = 'Decrypt'
    AND resources[1].arn = 'arn:aws:kms:...'
ORDER BY 1
```

There will also be `GenerateDataKey` events.

## Alerting

We recommend setting up alerts on suspicious behavior.

## Key Rotation

KMS supports [automatic key rotation](http://docs.aws.amazon.com/kms/latest/developerguide/rotate-keys.html). No action is required in this case.

To manually rotate keys, replace the old KMS key id with the new key id in your model. Your app does not need the old key id to perform rotation (however, the key must still be enabled in your AWS account).

```sh
KMS_KEY_ID=arn:aws:kms:...
```

and run

```ruby
User.find_each do |user|
  user.rotate_kms_key!
end
```

## IAM Permissions

A great feature of KMS is the ability to grant encryption and decryption permission separately.

To encrypt the data, use a policy with:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "EncryptData",
            "Effect": "Allow",
            "Action": "kms:GenerateDataKey",
            "Resource": "arn:aws:kms:..."
        }
    ]
}
```

If a system can only encrypt, you must clear out existing data and data keys before updates.

```ruby
user.encrypted_email = nil
user.encrypted_kms_key = nil
# before user.save or user.update
```

To decrypt the data, use a policy with:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DecryptData",
            "Effect": "Allow",
            "Action": "kms:Decrypt",
            "Resource": "arn:aws:kms:..."
        }
    ]
}
```

Be extremely selective of systems you allow to decrypt.

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

## History

View the [changelog](https://github.com/ankane/kms_encrypted/blob/master/CHANGELOG.md)

## Contributing

Everyone is encouraged to help improve this project. Here are a few ways you can help:

- [Report bugs](https://github.com/ankane/kms_encrypted/issues)
- Fix bugs and [submit pull requests](https://github.com/ankane/kms_encrypted/pulls)
- Write, clarify, or fix documentation
- Suggest or add new features
