![galaxy logo](../../docs/shared-images/galaxy_logo_25percent_transparent.png)

### Galaxy Administrators Course

# PAM Authentication in Galaxy - Exercise

#### Authors: Nate Coraor (2017), Nicola Soranzo (2018)

## Learning Outcomes

By the end of this session you should:
1. be familiar with Galaxy pluggable authentication framework and its configuration
2. have basic familiarity with the Linux PAM subsystem
3. be able to log in to your Galaxy server with a regular local user.

## Introduction

[Linux-PAM (Pluggable Authentication Modules)](http://www.linux-pam.org/) is a flexible mechanism for authenticating users on Linux.
PAM uses a modular architecture to provide a common authentication scheme for applications and services. PAM gives the system administrator significant flexibility and control over how individual applications will authenticate users.
PAM separates the tasks of authentication into four independent management groups: account management; authentication management; password management; and session management.

## Section 1 - Install dependencies

**Part 1 - Install python-pam**

For Galaxy pre-18.01, we need to install `python-pam` into Galaxy's virtualenv, `/srv/galaxy/venv`. To do this, run:

```console
$ sudo -Hu galaxy /srv/galaxy/venv/bin/pip install python-pam
Collecting python-pam
  Downloading python-pam-1.8.2.tar.gz
Building wheels for collected packages: python-pam
  Running setup.py bdist_wheel for python-pam ... done
  Stored in directory: /home/galaxy/.cache/pip/wheels/2a/37/e1/75c8e26b429857d73aeabd1fe39365c72a969706c30d9e6572
Successfully built python-pam
Installing collected packages: python-pam
Successfully installed python-pam-1.8.2
```

## Section 2 - Add a new local user account

We don't want to grant the `ubuntu` or `galaxy` user access to Galaxy, because we would be sending our VM admin or Galaxy user password in the clear. Instead, we create a new regular local user on the VM, `galaxyuser`, to log in with. If you were connecting Galaxy to your institution's authentication system, you wouldn't need to perform the steps in this section. Creating users in that system would be done via whatever mechanisms are appropriate for that system.

Create the user `galaxyuser` and set a nontrivial password that is not the same as `ubuntu`'s:

```console
$ sudo useradd -d /home/galaxyuser -m -s /bin/bash galaxyuser
$ sudo passwd galaxyuser
Enter new UNIX password:
Retype new UNIX password:
passwd: password updated successfully
```

You should now be able to see the new user with e.g. `getent`:

```console
$ getent passwd galaxyuser
galaxyuser:x:1001:1001::/home/galaxyuser:/bin/bash
```

One final system change needs to be made - again, only because we are using local user accounts for authentication (the ones in `/etc/passwd` and `/etc/shadow`). The Galaxy server user needs to have access to the `/etc/shadow` file where hashed passwords are stored. This is not something you should normally do as it is a small security risk (if a security flaw in Galaxy allowed reading arbitrary files, attackers could use it to read `/etc/shadow`).

This change adds the `galaxy` user to the `shadow` group, which allows it to read `/etc/shadow`, which has group ownership `shadow`:

```console
$ sudo usermod -a -G shadow galaxy
```

## Section 3 - Update Galaxy configuration

For this exercise we will use the `sshd` PAM service, which is configured in `/etc/pam.d/sshd`. You can have a look at this file and see how it works.


**Part 1 - Create a Galaxy auth config**

Create a new config file, `/srv/galaxy/config/auth_conf.xml`:

```console
$ sudo -u galaxy -e /srv/galaxy/config/auth_conf.xml
```

```xml
<?xml version="1.0"?>
<auth>
    <authenticator>
        <type>PAM</type>
        <options>
            <auto-register>True</auto-register>
            <maildomain>example.org</maildomain>
            <login-use-email>True</login-use-email>
            <pam-service>sshd</pam-service>
        </options>
    </authenticator>
</auth>
```

By default, Galaxy will look for this file in the `/srv/galaxy/server/config/` directory. We need to fix this in `galaxy.ini`:

```console
$ sudo -u galaxy -e /srv/galaxy/config/galaxy.ini
```

```ini
auth_config_file = /srv/galaxy/config/auth_conf.xml
```

Finally, restart Galaxy:

```console
$ sudo supervisorctl restart all
gx:handler0: stopped
gx:handler1: stopped
gx:galaxy: stopped
gx:handler0: started
gx:handler1: started
gx:galaxy: started
```

**Part 2 - Log in to Galaxy**

Go to http://yourgalaxyhost/ and log in (log out first if necessary) as `galaxyuser@example.org`.

**Part 3 - Optionally disable self registration and require login**

To configure Galaxy in this way, edit the galaxy config file using `sudo -u galaxy -e /srv/galaxy/config/galaxy.ini`:

```ini
require_login = True
allow_user_creation = False
```

Restart Galaxy with `sudo supervisorctl restart all`, then visit it in your browser. Log out and log back in again to see the results.

**Part 3 - Undo changes**

We don't want to leave Galaxy this way for the rest of our workshop. Undo the changes by removing the `auth_config_file`, `require_login` and `allow_user_creation` options from `galaxy.ini`, renaming `auth_conf.xml` using `sudo -u galaxy mv /srv/galaxy/config/auth_conf.xml /srv/galaxy/config/auth_conf.xml.pam` and restarting Galaxy again with `sudo supervisorctl restart all`.
