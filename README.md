# ansible-trainee
Example Ansible configuration to accompany a series of articles I am writing.

## Use
On a remote client use `ansible-pull` command to run the `local.yml` playbook, i.e. with a `sudo` user without password:

```shell
$ ansible-pull -U https://github.com/giobim/ansible-trainee.git -vv local.yml
```

If we need to pass the sudo passwor for the user we need to use the vault.yml which itself need the vault-password to open.
We also don't want put the vault password in clear text in this repository. In order to encrypt the sensitive file in this repository,
I use [BlackBox](https://github.com/StackExchange/blackbox). 

The BB scripts make it easy to decrypt them when you need to view or edit 
them, and decrypt them for use in production.
If you don't have a GPG key, set it up using, for example, [these instructions](https://help.github.com/articles/generating-a-new-gpg-key/).

Get in touch with the owner of this repository, he/her will  add your GPG public key and you will be allowed to decrypt those files.

## Decrypting using an automated user

An automated user (a "role account") is one that that must be able to decrypt without a passphrase. In general you'll want to do this for the user that pulls the files from the repo to the master. 

GPG keys have to have a passphrase. However, passphrases are optional on subkeys. Therefore, we will create a key with a passphrase then create a subkey without a passphrase.

We now will create the target gnupg keys on a secured host (yourself) for the NEWMASTER:

```shell
$ mkdir /tmp/$NEWMASTER
$ cd /tmp/$NEWMASTER
$ gpg --homedir . --gen-key
Your selection?
   (1) RSA and RSA (default)
What keysize do you want? (2048) DEFAULT
Key is valid for? (0) DEFAULT

# Real name: Ansible CI Deploy Account
# Email address: deployer@NEWMASTER.domain.name
```

Rather than a real email address, use the username@FQDN of the host the key will be used on. If you use this role account on many machines, each should have its own key. By using the FQDN of the host, you will be able to know which key is which. In this doc, we'll refer to username@FQDN as $KEYNAME

Save the passphrase somewhere safe! (I use lastpass)

Create a sub-key that has no password:

```shell
$ gpg --homedir . --edit-key deployer
gpg> addkey
(enter passphrase)
  Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
Your selection? 6
What keysize do you want? (3072)
Key is valid for? (0)
Command> key 2
(the new subkey has a "*" next to it)
Command> passwd
(enter the main key's passphrase)
(enter an empty passphrase for the subkey... confirm you want to do this)
Command> save
```

Securely export this directory to NEWMASTER:

```shell
gpg --homedir . --export -a deployer >$(pwd)/pubkey.txt
tar cvf /tmp/${NEWMASTER}_keys.tar .
rsync -avP /tmp/${NEWMASTER}_keys.tar $NEWMASTER.domain.name:/tmp/
```

On NEWMASTER, receive the new GnuPG config for the deployer account:

```shell
sudo -u deployer bash
mkdir -m 0700 -p ~/.gnupg
cd ~/.gnupg && tar xvf /tmp/keys.tar
```

Back on your secure machine, add the new email address to .blackbox/blackbox-admins.txt:

```shell
$ cd /path/to/the/repo
$ blackbox_addadmin $KEYNAME /tmp/NEWMASTER
$ git commit -m'NEW ADMIN: deployer@NEWMASTER.domain.name' .
```

Regenerate all encrypted files with the new key:

```shell
$ blackbox_update_all_files
$ git push
```

On NEWMASTER, clone and import the keys and decrypt the files:

```shell
$ git clone https://github.com/giobim/ansible-trainee.git
Cloning into 'ansible-trainee'...
remote: Enumerating objects: 120, done.
remote: Counting objects: 100% (120/120), done.
remote: Compressing objects: 100% (72/72), done.
remote: Total 120 (delta 46), reused 108 (delta 39), pack-reused 0
Receiving objects: 100% (120/120), 41.08 KiB | 2.42 MiB/s, done.
Resolving deltas: 100% (46/46), done.
demo@node-001:~$ cd ansible-trainee/
demo@node-001:~/ansible-trainee$ blackbox_postdeploy
========== Importing keychain: START
gpg: key 313068D83745977F: public key "Giovanni Binda (Ms. Sc., El. Engineer, Researcher, Globetrotter) <Giovanni.Binda@versatile.ch>" imported
gpg: Total number processed: 2
gpg:               imported: 1
gpg:              unchanged: 1
========== Importing keychain: DONE
========== Decrypting new/changed files: START
========== EXTRACTED dummy
========== EXTRACTED vault-password.txt
========== Decrypting new/changed files: DONE
demo@node-001:~/ansible-trainee$ 
```

If you get "gpg: decryption failed: No secret key" then you forgot to re-encrypt blackbox.yaml with the new key.

On your secured host, securely delete your files:

```shell
cd /tmp/NEWMASTER
# On machines with the "shred" command:
shred -u /tmp/keys.tar
find . -type f -print0 | xargs -0 shred -u
# All else:
rm -rf /tmp/NEWMASTER
```

Also shred any other temporary files you may have made.

