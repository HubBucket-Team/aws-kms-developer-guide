# How Amazon Elastic Transcoder Uses AWS KMS<a name="services-et"></a>

You can use Amazon Elastic Transcoder to convert media files stored in an Amazon S3 bucket into formats required by consumer playback devices\. Both input and output files can be encrypted and decrypted\. The following sections discuss how AWS KMS is used for both processes\.

**Topics**
+ [Encrypting the input file](#et-encrypt-input)
+ [Decrypting the input file](#et-decrypt-input)
+ [Encrypting the output file](#et-output)
+ [HLS Content Protection](#et-hls)
+ [Elastic Transcoder Encryption Context](#et-encryption-context)

## Encrypting the input file<a name="et-encrypt-input"></a>

Before you can use Elastic Transcoder, you must [create an Amazon S3 bucket](http://docs.aws.amazon.com/AmazonS3/latest/user-guide/CreatingaBucket.html) and upload your media file into it\. You can encrypt the file before uploading by using AES client\-side encryption or after uploading by using Amazon S3 server\-side encryption\.

If you choose client\-side encryption using AES, you are responsible for encrypting the file before uploading it to Amazon S3, and you must provide Elastic Transcoder access to the encryption key\. You do this by using an AWS KMS customer master key \(CMK\) to protect the AES encryption key you used to encrypt the media file\.

If you choose server\-side encryption, you are allowing Amazon S3 to perform all encryption and decryption of files on your behalf\. You can configure Amazon S3 to use one of three different master keys to protect the unique data key used to encrypt your file:
+ The Amazon S3 master key, a key that is owned and managed by AWS
+ The AWS\-managed CMK for Amazon S3, a master key that is owned by your account but managed by AWS
+ Any customer\-managed CMK that you create by using AWS KMS

You can request encryption and the master key you want by using the Amazon S3 console or the appropriate Amazon S3 APIs\. For more information about how Amazon S3 performs encryption, see [Protecting Data Using Encryption](http://docs.aws.amazon.com/AmazonS3/latest/dev/UsingEncryption.html) in the *Amazon Simple Storage Service Developer Guide*\.

When you use an AWS KMS CMK as the master key \(the AWS\-managed CMK for Amazon S3 in your account or a customer\-managed CMK\) to protect the input file, Amazon S3 and AWS KMS interact in the following manner:

1. Amazon S3 requests a plaintext data key and a copy of the data key encrypted under the specified CMK\.

1. AWS KMS creates a data key, encrypts it with the specified CMK, and then sends both the plaintext data key and the encrypted data key to Amazon S3\.

1. Amazon S3 uses the plaintext data key to encrypt the media file and then stores the file in the specified Amazon S3 bucket\.

1. Amazon S3 stores the encrypted data key alongside of the encrypted media file\.

## Decrypting the input file<a name="et-decrypt-input"></a>

If you choose Amazon S3 server\-side encryption to encrypt the input file, Elastic Transcoder does not decrypt the file\. Instead, Elastic Transcoder relies on Amazon S3 to perform decryption depending on the [settings you specify when you create a job](http://docs.aws.amazon.com/elastictranscoder/latest/developerguide/job-settings.html) and a pipeline\. 

The following combination of settings are available\.


****  

| Encryption mode | AWS KMS key | Meaning | 
| --- | --- | --- | 
| S3 | Default | Amazon S3 creates and manages the keys used to encrypt and decrypt the media file\. The process is opaque to the user\. | 
| S3\-AWS\-KMS | Default | Amazon S3 uses a data key encrypted by the default AWS\-managed CMK for Amazon S3 in your account to encrypt the media file\. | 
| S3\-AWS\-KMS | Custom \(with ARN\) | Amazon S3 uses a data key encrypted by the specified customer\-managed CMK to encrypt the media file\. | 

When **S3\-AWS\-KMS** is specified, Amazon S3 and AWS KMS work together in the following manner to perform the decryption\.

1. Amazon S3 sends the encrypted data key to AWS KMS\.

1. AWS KMS decrypts the data key by using the appropriate CMK, and then sends the plaintext data key back to Amazon S3\.

1. Amazon S3 uses the plaintext data key to decrypt the ciphertext\.

If you choose client\-side encryption using an AES key, Elastic Transcoder retrieves the encrypted file from the Amazon S3 bucket and decrypts it\. Elastic Transcoder uses the CMK you specified when you created the pipeline to decrypt the AES key and then uses the AES key to decrypt the media file\.

## Encrypting the output file<a name="et-output"></a>

Elastic Transcoder encrypts the output file depending on how you specify the encryption settings when you create a job and a pipeline\. The following options are available\.


****  

| Encryption mode | AWS KMS key | Meaning | 
| --- | --- | --- | 
| S3 | Default | Amazon S3 creates and manages the keys used to encrypt the output file\. | 
| S3\-AWS\-KMS | Default | Amazon S3 uses a data key created by AWS KMS and encrypted by the AWS\-managed CMK for Amazon S3 in your account\. | 
| S3\-AWS\-KMS | Custom \(with ARN\) | Amazon S3 uses a data key encrypted by using the customer\-managed CMK specified by the ARN to encrypt the media file\. | 
| AES\- | Default | Elastic Transcoder uses the AWS\-managed CMK for Amazon S3 in your account to decrypt the specified AES key you provide and uses that key to encrypt the output file\. | 
| AES\- | Custom \(with ARN\) | Elastic Transcoder uses the customer\-managed CMK specified by the ARN to decrypt the specified AES key you provide and uses that key to encrypt the output file\. | 

When you specify that an AWS KMS CMK \(the AWS\-managed CMK for Amazon S3 in your account or a customer\-managed CMK\) be used to encrypt the output file, Amazon S3 and AWS KMS interact in the following manner:

1. Amazon S3 requests a plaintext data key and a copy of the data key encrypted by using the specified CMK\.

1. AWS KMS creates a data key, encrypts it under the CMK, and sends both the plaintext data key and the encrypted data key to Amazon S3\.

1. Amazon S3 encrypts the media using the data key and stores it in the specified Amazon S3 bucket\.

1. Amazon S3 stores the encrypted data key alongside the encrypted media file\.

When you specify that your provided AES key be used to encrypt the output file, the AES key must be encrypted using a CMK in AWS KMS\. Elastic Transcoder, AWS KMS, and you interact in the following manner:

1. You encrypt your AES key by calling the [Encrypt](http://docs.aws.amazon.com/kms/latest/APIReference/API_Encrypt.html) operation in the AWS KMS API\. AWS KMS encrypts the key by using the specified CMK\. You specify which CMK to use when you are creating the pipeline\.

1. You specify the file containing the encrypted AES key when you create the Elastic Transcoder job\.

1. Elastic Transcoder decrypts the key by calling the [Decrypt](http://docs.aws.amazon.com/kms/latest/APIReference/API_Decrypt.html) operation in the AWS KMS API, passing the encrypted key as ciphertext\.

1. Elastic Transcoder uses the decrypted AES key to encrypt the output media file and then deletes the decrypted AES key from memory\. Only the encrypted copy you originally defined in the job is saved to disk\.

1. You can download the encrypted output file and decrypt it locally by using the original AES key that you defined\.

**Important**  
Your private encryption keys are never stored by AWS\. Therefore, it is important that you safely and securely manage your keys\. If you lose them, you won't be able to decrypt your data\.

## HLS Content Protection<a name="et-hls"></a>

HTTP Live Streaming \(HLS\) is an adaptive streaming protocol\. Elastic Transcoder supports HLS by breaking your input file into smaller individual files called media segments\. A set of corresponding individual media segments contain the same material encoded at different bit rates, thereby enabling the player to select the stream that best fits the available bandwidth\. Elastic Transcoder also creates playlists that contain metadata for the various segments that are available to be streamed\.

You can use AES\-128 encryption to protect the transcoded media segments\. When you enable HLS content protection, each media segment is encrypted using an AES\-128 encryption key\. When the content is viewed, the player downloads the key and decrypts the media segments during the playback process\.

Two types of keys are used: an AWS KMS CMK and a data key\. You must create a CMK to use to encrypt and decrypt the data key\. Elastic Transcoder uses the data key to encrypt and decrypt media segments\. The data key must be AES\-128\. All variations and segments of the same content are encrypted using the same data key\. You can provide a data key or have Elastic Transcoder create it for you\.

The CMK can be used to encrypt the data key at the following points:
+ If you provide your own data key, you must encrypt it before passing it to Elastic Transcoder\.
+ If you request that Elastic Transcoder generate the data key, then Elastic Transcoder encrypts the data key for you\.

The CMK can be used to decrypt the data key at the following points:
+ Elastic Transcoder decrypts your provided data key when it needs to use the data key to encrypt the output file or decrypt the input file\.
+ You decrypt a data key generated by Elastic Transcoder and use it to decrypt output files\.

For more information, see [HLS Content Protection](http://docs.aws.amazon.com/elastictranscoder/latest/developerguide/content-protection.html) in the *Amazon Elastic Transcoder Developer Guide*\.

## Elastic Transcoder Encryption Context<a name="et-encryption-context"></a>

An [encryption context](encryption-context.md) is a set of key–value pairs that contain arbitrary nonsecret data\. When you include an encryption context in a request to encrypt data, AWS KMS cryptographically binds the encryption context to the encrypted data\. To decrypt the data, you must pass in the same encryption context\. 

Elastic Transcoder uses the same encryption context in all AWS KMS API requests to generate data keys, encrypt, and decrypt\.

```
"service" : "elastictranscoder.amazonaws.com"
```

The encryption context is written to CloudTrail logs to help you understand how a given AWS KMS CMK was used\. In the `requestParameters` field of a CloudTrail log file, the encryption context looks similar to the following:

```
"encryptionContext": {
  "service" : "elastictranscoder.amazonaws.com"
}
```

For more information about how to configure Elastic Transcoder jobs to use one of the supported encryption options, see [Data Encryption Options](http://docs.aws.amazon.com/elastictranscoder/latest/developerguide/encryption.html) in the *Amazon Elastic Transcoder Developer Guide*\.