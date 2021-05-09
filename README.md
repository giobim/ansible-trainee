# ansible-trainee

Example Ansible configuration to accompany a series of articles I am writing.

## Use
On a remote client use `ansible-playbook` command to run the `local.yml` playbook, i.e. with a user that can us `sudo` without giving his/her password:

```shell
$ ansible-pull -U https://github.com/giobim/ansible-trainee.git -vv local.yml
```

Now, this command should be run by a user able to `sudo` without a password otherwise it would fails.

## Securing

Using a user that has sudo access without password is hihghly insecure. We can solve this issue using ansible vaults to store the sensitive information but we still need to persist the vault password itself. A way to solve this chicken-egg problem is to persist the vault password in the same repository and to use [BlackBox](https://github.com/StackExchange/blackbox) to secure this password. 

`BB` scripts make it easy to encrypt and decrypt them when you need to view or edit them. You also can decrypt them automatically for use in production without prompting you for a password.

 If you don't have a GPG key, set it up using, for example, [these instructions](https://help.github.com/articles/generating-a-new-gpg-key/) then, get in touch with the owner of this repository, he/her will add your GPG public key to allow you to decrypt those files.

## Decrypting in production using an automated user

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

## GIT Automation

When the client checks out the repository it should execute the `blackbox_postdeploy` command inside the repository.
A way to let GIT do this is to create a global folder holding the hook you want to execute.

However, this is a global setting which means it will get executed for every repository you are using on this host. This may be a problem if it was not limited to the per-user  configuration.

So we create a `.githooks/global` folder and add the `post-checkout`script:

```bash
#!/usr/bin/sh
/usr/bin/blackbox_postdeploy
```
We set the global config:

```shell
$ git config --global core.hooksPath ~/.githooks/global
```

We need to install balckbox on the host using the package for the platform and when all is in place, we can now test:

```shell
$ git clone -v -v https://github.com/giobim/ansible-trainee.git
Cloning into 'ansible-trainee'...
Server supports multi_ack_detailed
Server supports no-done
Server supports side-band-64k
Server supports allow-tip-sha1-in-want
Server supports allow-reachable-sha1-in-want
Server supports ofs-delta
Server supports filter
Server version is git/github-gb5447b43d7b7
want 02fd5be20e85a9dd391bf8960871433eef7c95d3 (refs/heads/main)
POST git-upload-pack (165 bytes)
done
remote: Enumerating objects: 159, done.
remote: Counting objects: 100% (159/159), done.
remote: Compressing objects: 100% (98/98), done.
remote: Total 159 (delta 62), reused 138 (delta 46), pack-reused 0
Receiving objects: 100% (159/159), 52.36 KiB | 2.75 MiB/s, done.
Resolving deltas: 100% (62/62), done.
========== Importing keychain: START
gpg: Total number processed: 3
gpg:              unchanged: 3
========== Importing keychain: DONE
========== Decrypting new/changed files: START
========== EXTRACTED dummy
========== EXTRACTED vault-password.txt
========== Decrypting new/changed files: DONE
```
Success!

## Putting all together
Ansible uses a vault to store the `demo` user password to get sudo access. The vault password is stored in a gpg encrypted file in the repository. It get decrypted by the above hook using a **password-less** public key **if found** in the user GNUPG tresor.
It is perfectly safe to use as far as the client has the necessary private key.

The last problem is that `ansible-pull` must find the vault password file somewhere in the repository this means the location given to `--vault-password-file` should be the full path. A simple solution is to set the destination of the checkout somewhere known at call.

Luckily, the modifier `--directory` allows to set where `ansible-pull` will put the repo.

Let's try it out:

```shell
$ ansible-pull -U https://github.com/giobim/ansible-trainee.git \
--directory=~/ansible-trainee \
--vault-password-file=~/ansible-trainee/vault-password.txt \
local.yml
Starting Ansible Pull at 2021-05-09 13:11:56
/usr/bin/ansible-pull -U https://github.com/giobim/ansible-trainee.git --directory=~/ansible-trainee --vault-password-file=~/ansible-trainee/vault-password.txt local.yml
 [WARNING]: Could not match supplied host pattern, ignoring: node-001.home.lab
 [WARNING]: Could not match supplied host pattern, ignoring: node-001
localhost | CHANGED => {
    "after": "0d8785eb4ee021bf3309d5f48ca365b7d013ed28",
    "before": null,
    "changed": true
}
 [WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'
 [WARNING]: Could not match supplied host pattern, ignoring: node-001.home.lab
 [WARNING]: Could not match supplied host pattern, ignoring: node-001

PLAY [Trainee playbook] ***********************************************************************************************************************************************************************

TASK [Gathering Facts] ************************************************************************************************************************************************************************
ok: [localhost]

TASK [update repositories] ********************************************************************************************************************************************************************
ok: [localhost]

TASK [Install packages] ***********************************************************************************************************************************************************************
changed: [localhost]

PLAY RECAP ************************************************************************************************************************************************************************************
localhost                  : ok=3    changed=1    unreachable=0    failed=0   

```

That's all, folks!