# Create a _perfect_ GnuPG Keys

Version 0.5 ([windows](https://github.com/bmone/gnupg-perfect-keys/tree/windows)) | Friday 3 January 2020

### Prepare the smartcard (security token)

Verify smartcard connection 
```
> gpg --card-status
```

Reset smartcard to factory defaults (optional)
```
> gpg --edit-card
  
  admin
  factory-reset
```

### Set smartcard default key attributes

I prefer modern ECC/Ed25519 over classic RSA key

|Sign|Encr|Auth|
|----|----|----|
|ed25519/sign|cv25519/encr|ed25519/auth)
```
> gpg-connect-agent "SCD SETATTR KEY-ATTR --force 1 22 ed25519" /bye
> gpg-connect-agent "SCD SETATTR KEY-ATTR --force 2 22 cv25519" /bye
> gpg-connect-agent "SCD SETATTR KEY-ATTR --force 3 22 ed25519" /bye
```

### Create a main key with subkeys (C-A-S-E)

|Key|Usage|Storage|Availabilty|
|---|---|---|---|
|Main|**C** _ertify_ |Secure backup|Offline |
|Subkey|**A** _uthenticate_|Smartcard|Online|
|Subkey|**S** _ign_ |Smartcard|Online|
|Subkey|**E** _ncrypt_ |Smartcard|Online|

The Ceritfy key would be stored offline, subkeys on a smartcard (security token)

Generate main **Certify** key
```
> gpg --expert --full-gen-key

   (11) ECC (set your own capabilities)
   (S) Toggle the sign capability [Leave Certify only]
   (Q) Finished
   (1) Curve 25519
   (4y) key expires in 4 years 
   Real name: <real name>
   Email address: <email address>
   Comment: <empty>
```

Assign environmental variable for main key id
```
> set MSTRFPR=<main key keyid> 
```

Edit main key
```
> gpg --expert --edit-key %MSTRFPR%
```

Add **Photo** id (optional ; keep the size around 240x288 or less) 
```
  addphoto
   Enter JPEG filename: <path to the photo file in jpg format>
```

Add **Authentication** subkey
```
  addkey
   (11) ECC (set your own capabilities)
   (A) Toggle the authenticate capability [Add Authenticate]
   (S) Toggle the sign capability [Remove Sign]
   (Q) Finished
   (1) Curve 25519
   (2y) key expires in 2 years
```

Add **Sign** subkey
```
  addkey
   (10) ECC (sign only)
   (1) Curve 25519
   (2y) key expires in 2 years
```

Add **Encryption** subkey
```
  addkey
   (12) ECC (encrypt only)
   (1) Curve 25519
   (2y) key expires in 2 years
```

Cross cerify signing subkey and save
```
  cross-certify
  save
```

Assign environmental variables main subkeys id
```
> set MSTRAUTHFPR=<main auth subkey keyid> 
> set MSTRSIGNFPR=<main sign subkey keyid> 
> set MSTRENCRFPR=<main encr subkey keyid> 
```

Generate revocation cerificates for main key (each for different reason)
```
> gpg --output %MSTRFPR%-revoke-cert-noreason.rev --armor --generate-revocation %MSTRFPR%
> gpg --output %MSTRFPR%-revoke-cert-compromised.rev --armor --generate-revocation %MSTRFPR%
> gpg --output %MSTRFPR%-revoke-cert-superseded.rev --armor --generate-revocation %MSTRFPR%
> gpg --output %MSTRFPR%-revoke-cert-nolongerused.rev --armor --generate-revocation %MSTRFPR%
```

Armor export main secret keys (including secret subkeys) [backup mode]
```
> gpg --output %MSTRFPR%-sec-keys-backup.asc --armor --export-options backup --export-secret-keys %MSTRFPR%
```

Armor export main secret key (excluding secret subkeys) [backup mode]
```
> gpg --output %MSTRFPR%-sec-cert-key-backup.asc --armor --export-options backup --export-secret-keys %MSTRFPR%!
```

Armor export main secret subkeys (excluding master secret key) [backup mode]
```
> gpg --output %MSTRFPR%-sec-subkeys-backup.asc --armor --export-options backup --export-secret-subkeys %MSTRFPR%
```

Armor export individual main secret subkeys [backup mode]
```
> gpg --output %MSTRFPR%-sec-subkey-auth-backup.asc --armor --export-options backup --export-secret-subkeys %MSTRAUTHFPR%!
> gpg --output %MSTRFPR%-sec-subkey-sign-backup.asc --armor --export-options backup --export-secret-subkeys %MSTRSIGNFPR%!
> gpg --output %MSTRFPR%-sec-subkey-encr-backup.asc --armor --export-options backup --export-secret-subkeys %MSTRENCRFPR%!
```

### Create primary certification key (C)

|Key|Key Usage|Storage|Availabilty|
|---|---------|-------|-----------|
|Primary|**C** _ertify_|Keyring|Online|

This Certify key would remain in key ring and would be designated by offline main certify key as a trusted introducer (crossed-certified)

Generate **Certify** key
```
> gpg --expert --full-gen-key
   
   (11) ECC (set your own capabilities)
   (S) Toggle the sign capability [Leave Certify only]
   (Q) Finished
   (1) Curve 25519
   (2y) key expires in 2 years 
   Real name: <real name> (certification key)
   Email address: <empty>
   Comment: <empty>
```

Assign environmental variable certify key id
```
> set CERTFPR=<certify key keyid> 
```

Edit key
```
> gpg --expert --edit-key %CERTFPR%
```

Add **Photo** id (optional)
```
  addphoto
   Enter JPEG filename: <path to the photo file in jpg format>
  save
```

Delegate certification key as trusted introducer for the main key
```
> gpg -u %MSTRFPR% --edit-key %CERTFPR%

  tsign
   (n) Signature to expire at the same time?
   (0) Signature does not expire
   (3) I have done very careful checking
   (2) I trust fully
   (2) Depth of this trust signature?
   Domain to restrict this signature? <empty>
  save
```

Cross certify the main key with certification key
```
> gpg -u %CERTFPR% --edit-key %MSTRFPR%
  
  sign
   (n) Signature to expire at the same time? No
   (0) Signature does not expire
   (3) I have done very careful checking
  save
```

Generate revocation cerificates for certification key (for each reason)
```
> gpg --output %CERTFPR%-revoke-cert-noreason.rev --armor --generate-revocation %CERTFPR%
> gpg --output %CERTFPR%-revoke-cert-compromised.rev --armor --generate-revocation %CERTFPR%
> gpg --output %CERTFPR%-revoke-cert-superseded.rev --armor --generate-revocation %CERTFPR%
> gpg --output %CERTFPR%-revoke-cert-nolongerused.rev --armor --generate-revocation %CERTFPR%
```

Armor export certification secret key [backup mode]
```
> gpg --output %CERTFPR%-sec-key-cert-backup.asc --armor --export-options backup --export-secret-keys %CERTFPR%
```

### Backup public keys

Armor export main public key (including public subkeys) [normal/backup mode]
```
> gpg --output %MSTRFPR%-pub-keys.asc --armor --export %MSTRFPR%
> gpg --output %MSTRFPR%-pub-keys-backup.asc --armor --export-options backup --export %MSTRFPR%
```

Export SSH authentication public key (main public subkey auth key)
```
> gpg --output %MSTRFPR%-openssh-id.pub --export-ssh-key %MSTRAUTHFPR%!
```

Armor export certification public key [normal/backup mode]
```
> gpg --output %CERTFPR%-pub-key.asc --armor --export %CERTFPR%
> gpg --output %CERTFPR%-pub-key-backup.asc --armor --export-options backup --export %CERTFPR%
```

### Prepare the main key for transfer to smartcard

Change password of the main secret key
```
> gpg --edit-key %MSTRFPR%
  
  passwd
  quit
```

Armor export main secret subkeys (excluding master secret key; changed password) [backup mode]
```
> gpg --output %MSTRFPR%-sec-subkeys-backup-changedpasswd.asc --armor --export-options backup --export-secret-subkeys %MSTRFPR%
```

Delete main public and secret keys
```
> gpg --delete-secret-and-public-keys %MSTRFPR%
```

Import main public key and secret subkeys (excluding main secret key; changed password) [backup mode]
```
> gpg --import %MSTRFPR%-pub-keys-backup.asc %MSTRFPR%-sec-subkeys-backup-changedpasswd.asc
```

Change imported main key trust to ultimate
```
> gpg --edit-key %MSTRFPR%
  
  trust
   (5) I trust ultimately
  quit
```
