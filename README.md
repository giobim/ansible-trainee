# ansible-trainee
Example Ansible configuration to accompany a series of articles I am writing.

## Use
On a remote client use `ansible-pull` command to run the `local.yml` playbook, i.e. a `sudo` user without password:

```shell
$ ansible-pull -U https://github.com/giobim/ansible-trainee.git -vv local.yml
```
