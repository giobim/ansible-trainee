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

Get in touch with the owner of this repository an add your public key as Admin and you will be ready to decrypt.

