---
layout: post
title:  "Sudo without password"
date:   2020-01-22 20:55:00 +0700
categories: linux
author: Muhammad Ardivan
tags: Linux/Unix,Security
---

How to run `sudo` command without prompting password:

1. Create backup of `/etc/sudoers`:
    ```bash
    $ sudo cp /etc/sudoers /etc/sudoers.bak
    ```

2. Then, using nano/vi to edit the `/etc/sudoers` file.
    ```bash
    $ sudo visudo
    ```
    By default visudo using `vi` editor, but if you want to use `nano` (for example).

    It will show similar like this and `enter` your preferred editor.
    ```bash
    $ sudo update-alternatives --config editor
    
    There are 4 choices for the alternative editor (providing /usr/bin/editor).
    Selection    Path                Priority   Status
    ------------------------------------------------------------
    * 0          /bin/nano            40        auto mode
    1            /bin/ed             -100       manual mode
    2            /bin/nano            40        manual mode
    3            /usr/bin/vim.basic   30        manual mode
    4            /usr/bin/vim.tiny    15        manual mode

    Press <enter> to keep the current choice[*], or type selection number: 
    ```

3. Add or Edit your `user` to add the `NO PASSWD` privilege. It will show similar like this.
   ``` bash
   
    # This file MUST be edited with the 'visudo' command as root.
    #
    # Please consider adding local content in /etc/sudoers.d/ instead of
    # directly modifying this file.
    #
    # See the man page for details on how to write a sudoers file.
    #
    Defaults    env_reset
    Defaults    mail_badpass
    Defaults    secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"

    # Host alias specification

    # User alias specification

    # Cmnd alias specification

    # User privilege specification
    root    ALL=(ALL:ALL) ALL

    # Members of the admin group may gain root privileges
    %admin ALL=(ALL) ALL

    # Allow members of group sudo to execute any command
    %sudo   ALL=(ALL:ALL) ALL

    # See sudoers(5) for more information on "#include" directives:

    #includedir /etc/sudoers.d
    $USER ALL=(ALL) NOPASSWD: ALL # $USER is your user in Unix/Linux host.
   ```

4. To give it a try, you can save the new `/etc/sudoers/` file and do:
    ```bash
    $ sudo su
    # root@your-pc-name:/home/user# 
    ```