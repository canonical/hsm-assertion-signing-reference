# Snapd External Key Manager Support

## Overview

This document describes how [snapd](https://github.com/snapcore/snapd) can be configured to use an external key manager to support use of cryptographic keys stored on Smart Cards, Crypto Tokens, or HSMs to sign `model` and `system-user` assertions. For the purposes of this document, the term Crypto Device will be used.

**Note** - Guidance on selecting a specific Crypto Device is outside the scope of this document.

### Snap Keys

Snapd uses `rsa:4096` keys for signing assertions. Signing keys are normally created by a developer using the brand account via the `snapcraft create-key` command. Snapcraft stores its keys in a separate OpenPGP keyring located by default in the directory `$HOME/.snap/gnupg`.

Keys used to sign `model` and `system-user` assertions need to be registered to the brand account in the Snap Store via the `snapcraft register-key` command.

### External Key Manager
Snapd supports use of an external key manager for registering keys, and signing assertions. The actual provisioning of the signing key to the Crypto Device is not supported by snapd, and thus must be managed separately.

The process to use an external key manager is initiated by setting the environment variable `SNAPD_EXT_KEYMGR` prior to using the required `snap` or `snapcraft` commands. This environment variable must be set to the fully qualified path of a wrapper script (which must be made executable) which provides the integration between snapd and the Crypto Device. 

### Reference Implementation
Our reference implementation of an external key manager is a shell script called `pkcs11-snap-wrapper`. It uses the [OpenSC](https://github.com/OpenSC/OpenSC/wiki) project's `pkcs11-tool` to integrate with a PKCS11 provider, which may support one or more Crypto Devices. The OpenSC project is one such provider, and is in fact the default provider for the `pkcs11-tool`.

The external key manager must support:
- PKCS-RSA signing
- Public key DER output

The key manager must provide implementations for the following commands (see reference implementation for more details):
- `features`
  - output: `{"signing": ["RSA-PKCS"], "public-keys": ["DER"]}`
- `key-names`
  - output: `{"keynames": ["key-name-1", "key-name-2"]}`
- `get-public-key -f DER -k <key-name>`
  - output: the key, to stdout
- `sign -m RSA-PKCS -k <key-name>`
  - input: assertion to sign, from stdin
  - output: signed assertion, to stdout

**Note** - the `pkcs11-tool` can also be used with other PKCS11 providers via it's `--module` option, which specifies the fully qualified path to another PKCS11 provider (e.g. `softhsm2`).

As mentioned previously, the actual provisioning of the signing key must happen before the external key manager is used. In general, there are two approaches:

- Create the key using `gpg` and then import the key into the Crypto Device.
  - This approach allows for a signing key to be provisioned to more then one Crypto Device, providing backups of the key.
  - If this technique is used, care must be taken to ensure that local copies of the private key are shredded after import.
- Use `pkcs11-tool` (or other tool provided by the Crypto Device vendor) to generate a new key within the Crypto Device.
  - This ensures that the private key never leaves the Crypto Device.

#### Requirements

The reference solution has the following requirements:
- A computer running a recent Ubuntu Desktop LTS release (e.g. 20.04 or 22.04)
- A Crypto Device which is supported on Ubuntu via the `pkcs11-tool` provided by the OpenSC package

**Note** - this reference implementation has been tested with a [Nitrokey HSM](https://shop.nitrokey.com/shop/product/nkhs2-nitrokey-hsm-2-7)

#### System User Assertions
It also should be noted that the [`make-system-user` snap]((https://snapcraft.io/make-system-user)) does not support use of an external key manager.  As such, we've provided an additional shell script called `sign-system-user-assertion` which can be used with the reference external key manager to create a signed `system-user` assertion.

This script must be run while signed into snapd as the brand account, so that `account` and `account-key` assertions can be retrieved from the Snap Store and output along with the signed `system-user` assertion. All three assertions are required to trigger creation of an actual user account on the system.

**Note** - this script cannot be used if the `system-user-authority` specified in the `model` assertion differs from the brand authority. 
