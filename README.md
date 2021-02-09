# Yubikey as an SSH key

**As of 2020-05-09 [Filippo Valsorda](https://filippo.io/) has released [yubikey-agent](https://github.com/FiloSottile/yubikey-agent). I am now recommending this method over using PKCS#11, however if you still wish to use the native ssh-agent, read on.**

All other guides I've seen (https://github.com/drduh/YubiKey-Guide being the most prolific) tell you to use the Yubikey's smartcard (PKCS#11) features with GnuPG via gpg-agent.

STOP THE MADNESS!

OpenSSH has supported OpenSC since version 5.4. This means that all you need to do is install the OpenSC library and tell SSH to use that library as your identity.

## Prequisites

### 1. Install OpenSC and YubiKey Manager (CLI only)

#### On macOS
Ensure you install the cask version of OpenSC, not the formula. The cask version is a .pkg which will install the shared library to a location acceptable by `ssh-agent`. The formula does not, as Homebrew installs each version into its own location and it won't allow an unknown path to be used as a PKCS#11 library.

```sh
brew install --cask opensc
brew install ykman   
```

#### On Ubuntu/Debian
```
sudo apt-add-repository ppa:yubico/stable
sudo apt update
sudo apt install opensc yubikey-manager
```

### 2. If this is a new Yubikey, change the default PIV management key, PIN and PUK.

The `ykman` tool can generate a new management key for you. For the PIN and PUK you'll need to provide your own values (6-8 digits).

```
ykman piv change-management-key --touch --generate
ykman piv change-pin -P 123456
ykman piv change-puk -p 12345678
```

Make sure you save the generated password somewhere secure such as a password manager. The management key is needed any time you generate a keypair, import a certificate or change the number of PIN or PUK retries

The PUK should also be kept somewhere safe. This is used if the PIN is entered incorrectly too many times.

## How?

I did this all on macOS 10.14. Linux distributions should work in a similar way. This is based on [Yubico's instructions](https://developers.yubico.com/PIV/Guides/SSH_with_PIV_and_PKCS11.html) but uses the newer `ykman` utility instead of the older `yubico-piv-tool`. The older tool doesn't seem to support generating PIV certificates and gives [misleading errors](https://github.com/Yubico/yubico-piv-tool/issues/153).


1. Ensure CCID mode is enabled on the Yubikey

```
ykman mode
```

If CCID is not in the list, enable it by adding CCID to the list, e.g.

```
ykman mode OTP+FIDO+CCID
```

(This assumes you had OTP+FIDO previously, and still want them enabled.)

2. Generate a PIV key and output the public key

```
ykman piv generate-key 9a pubkey.pem
```

Alternatively, you can require that you have to touch the Yubikey every time the slot is accessed:

```
ykman piv generate-key --touch-policy always 9a pubkey.pem
```

This is an RSA 2048-bit key by default. Depending which Yubikey you have, you can change it using `-a` / `--algorithm`.

(9a is the PIV authentication slot.)

3. Generate a self-signed X.509 certificate

```
ykman piv generate-certificate -s "SSH key" 9a pubkey.pem
```

4. Export your SSH public key from the Yubikey

```
ssh-keygen -D /usr/local/lib/opensc-pkcs11.so
```

And that's all the hard stuff done. 

Now just add the public key to your `authorized_keys` file on a remote host and try to use it:

```
ssh -I /usr/local/lib/opensc-pkcs11.so -i /usr/local/lib/opensc-pkcs11.so -o IdentitiesOnly=yes server.example.com
```

You should be prompted for your Yubikey's PIV PIN.

You can add the PKCS11 library to `ssh-agent`.

```
ssh-add -s /usr/local/lib/opensc-pkcs11.so
```

Once more you will be prompted for your PIN, and from there SSH authentication will happen as usual.

To configure `ssh` to use the Yubikey's SSH key, use the `PKCS11Provider` config option instead of `IdentityFile`, e.g.:

```
Host foo
  PKCS11Provider /usr/local/lib/opensc-pkcs11.so
  IdentitiesOnly yes
````

## Additional notes

- When SSHing, you may get prompted with the key's subject name, like `Enter PIN for 'SSH key':`. But if you add the key to the agent, you'll get a prompt like `Enter passphrase for PKCS#11:`. These are the same PIN (your PIV PIN).

- If you remove the key from ssh-agent using `ssh-add -d` or `ssh-add -D`, you'll have to either remove and re-add the PKCS library to the agent or restart the agent. 
  - To re-add the library run
    ```
    ssh-add -e /usr/local/lib/opensc-pkcs11.so
    ssh-add -s /usr/local/lib/opensc-pkcs11.so
    ```
  - On macOS, you can restart the agent with
    ```
    launchctl stop com.openssh.ssh-agent
    launchctl start com.openssh.ssh-agent
    ```
