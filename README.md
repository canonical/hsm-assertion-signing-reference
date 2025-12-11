# External key manager support

Snapd and Snapcraft can leverage a key manager in conjunction with a hardware
security module (HSM) or smart card (SC) (collectively, a crypto device) to
sign assertions.

This is a reference implementation of a key manager using a [Nitrokey HSM](https://shop.nitrokey.com/shop/product/nkhs2-nitrokey-hsm-2-7),
but any crypto device satisfying the requirements should be sufficient.

## Requirements

The crypto device must support:

- PKCS-RSA signing
- Public key DER output

The key manager must implement features aligning with snapd's [external key manager API](https://github.com/canonical/snapd/blob/master/asserts/extkeypairmgr.go).
See the reference implementation script for an example.

The reference implementation is intended to be used on an Ubuntu LTS release
with the pkcs11-tool provided by the opensc package.

## Keys

Snapd uses `rsa:4096` keys for signing assertions. Signing keys are normally
created with the `snapcraft create-key` command. Snapcraft stores keys
generated this way in a separate OpenPGP keyring located in the directory
`$HOME/.snap/gnupg` by default.

### Using an external key manager

Snapd and Snapcraft can be made to use an external key manager rather than
the on-disk keyring for signing assertions by setting an environment variable,
`SNAPD_EXT_KEYMGR`. This variable must be set to the fully qualified path of the
external key manager implementation.

For example, to use the reference implementation downloaded to `$HOME/.local/bin`,

```bash
  export SNAPD_EXT_KEYMGR="$HOME/.local/bin/pkcs11-snap-wrapper"
```

### Limitations

Snapd and Snapcraft cannot currently use an external key manager for key creation.

See [key creation](#key-creation) for instructions compatible with the
reference implementation.

## Reference implementation

The reference implementation of an external key manager is a shell script called
`pkcs11-snap-wrapper`. It uses the [OpenSC](https://github.com/OpenSC/OpenSC/wiki)
project's `pkcs11-tool` to integrate with a PKCS11 provider, which may support
one or more crypto devices. The OpenSC project is one such provider, and is the
default provider for the `pkcs11-tool`.

The `pkcs11-tool` can also be used with other PKCS11 providers via it's
`--module` option, which specifies the fully qualified path to another PKCS11
provider (e.g. `softhsm2`). This option is not implemented in the reference.

## Key creation

The actual provisioning of the signing key must happen before the external key
manager is used. In general, there are two approaches:

- Use `pkcs11-tool` (or another tool provided by the crypto device vendor) to
  generate a new key within the crypto device.
  - This ensures that the private key never leaves the crypto device.
- Create the key using `gpg` and then import the key into the crypto device.
  - This approach allows for a signing key to be provisioned to more than one
    crypto device, providing backups of the key.
  - If this technique is used, care must be taken to ensure that local copies of
    the private key are shredded after import.

### Prerequisites

First install some prerequisite packages. On an Ubuntu LTS:

```bash
  apt install gnupg-pkcs11-scd         \
              libengine-pkcs11-openssl \
              libp11-kit0 opensc       \
              pcsc-tools pcscd
```

If you have only one HSM, it will be available to `pkcs11-tool` in slot 0. If
you have multiple HSMs plugged in, be sure to specify the correct `--slot`.
You can see all occupied slots with `pkcs11-tool --list-slots`. The reference
implementation does not support specifying a `--slot`, so when signing
assertions ensure the correct HSM is available on slot 0.

To start you need to initialize the HSM and choose some PINs.

You may or may not need to pass `--module path/to/opensc-pkcs11.so`. The usual
path on Ubuntu is `usr/lib/<arch triplet>/opensc-pkcs11.so`.

To (re)initialize the card, the current Security Officer (SO) PIN must be used.
If you've never initialized the card before or you used the same PIN as it came
with, you're using the dummy SO/user PINs.

The below will wipe anything and everything currently on the HSM. Choose
whatever label you like.

```bash
  pkcs11-tool --init-token --init-pin \
    --so-pin=3537363231383830         \
    --new-pin=648219 --pin=648219     \
    --label="harpocrates"
```

The provided pin codes are the dummy/default ones, and SHOULD be changed.

The SO PIN **must** be sixteen hexadecimal characters. To change the SO PIN:

```bash
  pkcs11-tool --login --login-type so \
    --so-pin 3537363231383830         \
    --change-pin --new-pin 0123456789abcdef
```

Anyone can change their non-SO (user) pin. User PINs can be up to sixteen
hexidecimal characters, but it is sufficient to choose six. To change the user
PIN:

```bash
  pkcs11-tool --login --pin 648219 \
    --change-pin --new-pin 123456
```

If you enter the user PIN incorrectly three times, you get locked out.
To unblock and change the PIN, you need the SO-PIN:

```bash
  pkcs11-tool --login --login-type so \
    --so-pin=3537363231383830         \
    --init-pin --new-pin=648219
```

All references to the SO and user PINs here will be the dummy PINs.

### Generating keys

To generate a key pair, use the below command. You may omit ID and label. If the
key is intended to be used for assertion signing, the key must be rsa:4096.

```bash
  pkcs11-tool --login --pin 648219   \
    --keypairgen --key-type rsa:4096 \
    --id 10 --label model

  Using slot 0 with a present token (0x0)
  Key pair generated:
  GnuPG Object; RSA 
    label:      model
    ID:         10
    Usage:      decrypt, sign, signRecover
    Access:     sensitive, always sensitive, never extractable, local
    uri:        pkcs11:model=PKCS%2315%20emulated;manufacturer=www.CardContact.de;serial=DENK0301274;token=harpocrates;id=%10;object=GnuPG;type=private
  Public Key Object; RSA 4096 bits
    label:      model
    ID:         10
    Usage:      encrypt, verify, verifyRecover
    Access:     none
    uri:        pkcs11:model=PKCS%2315%20emulated;manufacturer=www.CardContact.de;serial=DENK0301274;token=harpocrates;id=%10;object=GnuPG;type=public
```

To list the (public) key, you don't need to login (`--login --pin <pin>`), but
to list the private key you do.

### Registering the key

Now that a key exists on the HSM, you should be able to register the key to your
Ubuntu One SSO account in the usual way. First, download the pkcs11-snap-wrapper
reference implementation script and put it somewhere like `$HOME/.local/bin`,
and export the environment variable:

```bash
  mkdir -p "$HOME/.local/bin"
  curl -fLo "$HOME/.local/bin/pkcs11-snap-wrapper" https://raw.githubusercontent.com/canonical/hsm-assertion-signing-reference/refs/heads/main/pkcs11-snap-wrapper
  export SNAPD_EXT_KEYMGR="$HOME/.local/bin/pkcs11-snap-wrapper"
  export PKCS11_PIN=<pin>
```

Now you can use the `snapcraft register-key model` command to register the
`model` key you just created to your Ubuntu One SSO account.

### Verifying

To verify that the key can be used for signing, login to your Ubuntu One SSO
account and create a generic model assertion to sign:

```bash
  ACCOUNTID="$(snapcraft whoami | awk '($1=="id:") {print $2}')"

  cat << EOF > model.json
  {
    "type": "model",
    "authority-id": "$ACCOUNTID",
    "brand-id": "$ACCOUNTID",
    "series": "16",
    "model": "model",
    "architecture": "amd64",
    "base": "core24",
    "grade": "dangerous",
    "timestamp": "$(date -Iseconds --utc)",
    "snaps": [
      {
          "name": "pc",
          "type": "gadget",
          "default-channel": "24/stable",
          "id": "UqFziVZDHLSyO3TqSWgNBoAdHbLI4dAH"
      },
      {
          "name": "pc-kernel",
          "type": "kernel",
          "default-channel": "24/stable",
          "id": "pYVQrBcKmBa0mZ4CCN7ExT6jH8rY1hza"
      },
      {
          "name": "core24",
          "type": "base",
          "default-channel": "latest/stable",
          "id": "dwTAh7MZZ01zyriOZErqd1JynQLiOGvM"
      },
      {
          "name": "snapd",
          "type": "snapd",
          "default-channel": "latest/stable",
          "id": "PMrrV4ml8uWuEUDBT8dSGnKUYbevVhc4"
      },
      {
          "name": "console-conf",
          "type": "app",
          "default-channel": "24/stable",
          "id": "ASctKBEHzVt3f1pbZLoekCvcigRjtuqw",
          "presence": "optional"
      }
    ]
  }
  EOF

    snapcraft list-keys
      Name          SHA3-384 fingerprint
  *   model         <fingerprint>

  snap sign -k model model.json > model.assert
```

If it worked, a valid and signed model assertion should be created and can be
used to create images:

```bash
  ubuntu-image snap model.assert
```
