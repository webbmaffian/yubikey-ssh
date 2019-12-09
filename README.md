# How to use a Yubikey with SSH on Mac
Using a Yubikey for connecting from a Mac to a server via SSH wasn't so straight-forward as it seemed. You have probably read [drduh's guide](https://github.com/drduh/YubiKey-Guide) or even [this one](https://github.com/jamesog/yubikey-ssh), but netiher of them is straight-forward while supporting Yubico's recommendation of having two Yubikeys (one primary, one backup).

After this guide, you will have **one primary key** for everyday SSH use, and **one backup key** to put in a safe, both sharing **the same private key**, without GPG/PGP.

This is a step-by-step guide for how to succeed.

## Prerequisites
- MacOS with Homebrew
- 2 x Yubikeys (we use 5C, but any CCID-enabled Ubikey should work)

## Configure the Ubikeys
These steps should preferably be done on a fresh computer that we easily can disconnect from internet, and earse the whole computer afterwards. We used a Raspberry Pi. It might sound paranoid, but let's just be 100% sure that your private key can't be compromised in any way. If you follow this recommendation or not is totally up to you.

### 1. Install YubiKey Manager
Either:

#### a. If you're on MacOS:
```
brew install ykman
```

#### b. If you're on Ubuntu/Debian/Raspbian
```
sudo apt-get update
sudo apt-get install yubikey-manager
```

### 2. Go offline
I'm not kidding - disconnect from internet.

### 3. Setup Management Key (repeat per Ubikey)
Connect your Ubikey, and either:

#### a. Generate a key (ensure to save the output key)
```
ykman piv change-management-key --touch --generate
```

#### b. Set a key manually
```
ykman piv change-management-key --touch
```

### 4. Change number of allowed PIN/PUK retries (optional)
The default is 3 retries each, then you're screwed. You might want to increase these:
```
ykman piv set-pin-retries 10 10
```

### 5. Change PIN/PUK codes
The default PIN/PUK codes are `123456` and `12345678` respectively - we do obviously want to change these:
```
ykman piv change-pin
ykman piv change-puk
```

### 6. Generate RSA key pair
Yubikey 5C does unfortunately only support 2048-bit RSA keys at best.
```
openssl genrsa -des3 -out key.pem 2048
openssl rsa -in key.pem -outform PEM -out private.pem
openssl rsa -in key.pem -outform PEM -pubout -out public.pem
```

### 7. Import RSA private key to Ubikey (repeat per Ubikey)
Requiring key touch will improve security, but we don't want to do this all the time. Setting `touch-policy` to `cached` will "only" require a touch every 15 seconds. You can also choose `never` if you're bothered with this.
```
ykman piv import-key --touch-policy cached 9a private.pem
ykman piv generate-certificate -s ssh 9a public.pem
```

### 8. Save the key files to a USB stick
As the private keys on Yubikeys are read-only, you should really want to save the generated files to an encrypted USB stick. If you don't want to bother with encrypted partitions, the easiest encryption is a ZIP file. It's better than nothing:
```
zip -e piv.zip .
```

### 9. Clear your computer & lock away your devices
If it's a Raspberry Pi, earse everything on the SD card. If not, then ensure all files are properly deleted permanently and empty your trash can. Put your USB stick and backup Yubikeys in a safe.

## Configure MacOS
Now we're back online with our MacOS. Let's configure the ssh-agent to listen for our Ubikeys.

### 1. Install OpenSC
```
brew cask install opensc
ssh-add -s /usr/local/lib/opensc-pkcs11.so
```

### 2. Export RSA public key
These should be identical for every key:
```
ssh-keygen -D /usr/local/lib/opensc-pkcs11.so
```

Add it to the servers you want to SSH into with your Ubikey.

### 3. Configure SSH
Edit your `~/.ssh/config` file with either:

#### a. Use Ubikey for all hosts
```
Host *
  PKCS11Provider /usr/local/lib/opensc-pkcs11.so
  IdentitiesOnly yes
```

#### b. Use Ubikey for specific host
```
Host example.com
  PKCS11Provider /usr/local/lib/opensc-pkcs11.so
  IdentitiesOnly yes
```

## Usage
You should now be able to connect to your servers as usual as long as your Ubikey is connected:
```
ssh username@host.com
```

### If you required touch
You will notice that when you connect to the server, it will freeze. This is because the key is waiting for your touch. Touch it, and you're in.
