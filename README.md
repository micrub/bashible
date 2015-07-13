# BASHIBLE

Bashible is a deployment/automation tool written in Bash, inspired by Ansible. 

Bashible blebook is a better readable bash script.

## Why?

Tools like Puppet, Chef or Ansible are sometimes too heavy to do simple things. If you are in a hurry, there's no time to write multiple lines of code to achieve a copy of a single file. On the other hand, without any tools, shell scripts tend to become difficult to read and usually they are also full of unhandled failures.

## Example

An example blebook:

```bash
@ Adding user webuser and his public key
  - may_fail useradd webuser
  - add_line "$PUBLIC_KEY" /home/webuser/.ssh/authorized_keys

@ Running bundle install
  - cd /var/www
  - as webuser bundle install

@ Install nginx unless already
  - already_if test -x /usr/sbin/nginx
  - yum_install nginx
  - rsync -av /adm/config/webserver/ /

@ Start nginx and ensure it is running
  - service nginx start
  - timeout 20 wait_for_tcp 10.0.3.188:80 up
```

The very same in Bash:

```bash
basedir=`dirname $(readlink -f "$0")`

cd "$basedir" || { echo "can't chdir"; exit 1; }
echo "Adding user webuser and his public key"
useradd webuser
if ! grep "$PUBLIC_KEY" /home/webuser/.ssh/authorized_keys; then
   echo "$PUBLIC_KEY" >> /home/webuser/.ssh/authorized_keys || { echo "can't edit file"; exit 1; }
fi

cd "$basedir" || { echo "can't chdir"; exit 1; }
echo "Running bundle install"
cd /var/www || { echo "can't chdir"; exit 1; }
sudo -u webuser bundle install || { echo "can't bundle install"; exit 1; }

cd "$basedir" || { echo "can't chdir"; exit 1; }
if [ ! -x /usr/sbin/nginx ]; then
  echo "Installing nginx unless already"
  yum install -y nginx || { echo "yum failed"; exit 1; }
  echo "Installing default vhosts.d contents unless already"
  rsync -av /adm/config/webserver/ / || { echo "rsync failed"; exit 1; }
fi

echo "Start nginx and ensure it is running"
service start nginx
timeout 20 bash -c '
  while ! netstat -ltn | grep LISTEN | grep 10.0.3.188:80; do
    sleep 1; 
  done
'
```

`bashible blebook.ble` would then run the blebook, but it is just an example. Really working examples will come later.

## Install & usage

```bash
 cd /usr/local/bin
 wget https://raw.githubusercontent.com/mig1984/bashible/master/bashible
 chmod 755 bashible
```

 then

```bash
 bashible your-blebook.ble
```

## How does it work?

  - Bashible blebook is a better readable bash script.
  - There are blocks **"@ A comment..."** and tasks **"- command ..."**
  - Blocks have two purposes:
     - **skipping tasks** that already have been done
     - the **working directory** of each block starts in the same directory as the blebook resides, unless you explicitly specify elsewhere
  - If a **task fails** => the complete run of the **blebook fails**.
  - You can use **environment variables** to modify the run of a blebook (skip tasks, etc.)
  - You can **call** another blebook(s) from a blebook (like dependencies), moreover bashible doesn't call the same blebook twice (unless forced).
  - Bashible implements **helper functions** to simplify common tasks (e.g. adding lines to files, commenting lines in files, etc.)
  - You can use your **own helpers** (importing functions).
  - Finally, you can use a bash code if you need something special.

  At the moment it is used on Linux (Centos) only. May not work on other platforms (sed, netstat, etc. may behave differently elsewhere).

  Suggestions and patches are welcome! :-)


### Description of core functions

##### already

a) If used before tasks started, skips execution of all tasks in the blebook (exits the process).

```bash
when 'test -x /usr/sbin/nginx' already
```

b) If used inside a tasks block, skips following tasks up to the next block.

```bash
@ Installing nginx unless already
  - when 'test -x /usr/sbin/nginx' already
  - yum install nginx
```

note: does the same as the "skip", but outputs a different message

##### already_if COMMAND ...

Runs the command and if it results in true, calls "already"

a) before task blocks started

```bash
already_if test -x /usr/sbin/nginx
```

b) inside a tasks block

```bash
@ Installing nginx unless already
  - already_if test -x /usr/sbin/nginx
  - yum install nginx
```


##### as USER COMMAND ...

Run the command as the user. Sudo in non-login shell mode is used (the current working dir should be preserved).

notice: only some environment variables are copied to the sudoed command (e.g. PATH), other depends on your sudoers config (the sudo may clear the environment).

```bash
@ Installing dependencies of rails app
  - cd /home/rails
  - as www-data bundle install
```

##### base_dir PATH

Change a base working directory. On each tasks block the working directory is (re)set to the base directory.
Unless set, defaults to the same directory as the blebook resides in.

The reset_base_dir resets it to the default.

```bash
base_dir "/home/user1"

@ Doing something in /home/user1
- foo
- bar

...

reset_base_dir
```

##### call BLEBOOK_PATH [ARG1] [ARG2] ...

Call another blebook. It will run in a new process, therefore it won't affect the caller anyhow.
The working directory of the called blebook is set to the same directory as the blebook file resides in.
The "call" is used also for the top blebook.

The same blebook won't run again if it has already been called (use 'force_call' if you want to bypass the check).

```bash
call "./another-blebook.ble"
call "./another-blebook.ble" "arg1" "arg2"
```

see also 'force_call' which runs always again
see also 'notify_call' for postponed calls

##### fail MESSAGE

Interrupts execution with a message displayed in red on stdout.

```bash
- when 'test -x /usr/sbin/nginx' fail "nginx must be installed first"
```

##### force_call BLEBOOK_PATH [ARG1] [ARG2] ...

The same as 'call' but bypasses the already-called check (always calls the blebook).

```bash
force_call "./another-blebook.ble"
force_call "./another-blebook.ble" "arg1" "arg2"
```

##### import PATH

Use "import" instead of "source". It does the same, but remembers imported files. When a command is run using sudo, all files will be re-sourced
(otherwise functions won't be reachable in the child process).

```bash
import '/usr/local/share/mylib.sh'
```

##### i_am_child

Return true if the currently running blebook is called from another blebook.

```bash
unless 'i_am_child' fail "This blebook can't be called directly"
```

##### only_on HOST HOST ...

a) If used before task blocks, stops execution of the blebook unless it is not run on one of specified hosts (only_on).

```bash
only_on web123
```

b) If used inside a task block, skips following tasks unless it is not run on one of specified hosts (only_on).

```bash
@ Installing nginx
  - only_on web123 web125 web126
  - ...
```

The "not_on" does the opposite.

##### nonempty COMMAND ...

Runs the command and fails if it doesn't produce any output (or also fails).

In this example it loads domain.txt into a variable "domain" but fails if the file is not present or is empty.

```bash
set_var domain nonempty cat /etc/myapp/domain.txt
```

##### not COMMAND ...

Runs the command and negate it's exit status.

```bash
@ Doing cleanup
  - already_if not test -f '/tmp/foobar'
```

(or use "when" or "unless" with an exclamation mark)


##### notify_call BLEBOOK_PATH

Works like the 'call', but only remembers what should be called. The blebook will be called at the end of the process
and will be called only once. 

For instance: multiple blebooks want to restart nginx, but the restart should happen only once in the end, not each time.

```bash
notify_call './restart-nginx.ble'
```

##### not_on HOST HOST ...

see the "only_on" above (does the opposite)

##### may_fail COMMAND ...

Normally if a command fails, execution of a blebook is stopped.
Sometimes tasks may fail and it doesn't matter.

```bash
@ Cleaning up
  - may_fail rmdir /home
  - may_fail rmdir /var
```

##### quiet COMMAND ...

Runs the command with output directed to /dev/null.

```bash
- when 'quiet grep centos /etc/hosts' already
```

##### reset_base_dir

see the "base_dir" above


##### set_var NAME [COMMAND...] #####

Store the command's output in the variable. Fails if the command fails.

```bash
set_var list ls /
```

You can use more complex command using 'evaluate' (which also fails if the evaluated string fails).

```bash
set_var list evaluate "ls / | grep usr"
```

You can make even more complex chain, for instance, there is a value stored in a file which we'd like to load into a variable. Moreover it should fail if the loaded value is empty.

```bash
set_var myuser nonempty evaluate 'cat /etc/hosts | grep myuser'
```


##### skip

a) if used before all task blocks started, skips all following tasks in the blebook (exits the process)

```bash
when 'test $ENVIRONMENT = development' skip
```

b) skips following tasks up to a next block

```bash
@ Installing application if not in development mode
  - when 'test $ENVIRONMENT = development' skip
  - yum install myapp
```

note: does the same like the "already", but outputs a different message

##### skip_if COMMAND ...

Runs the command and if it results in true, calls "skip"

a) before task blocks started

```bash
skip_if 'test $ENVIRONMENT = development'
```

b) inside a tasks block

```bash
@ Installing nginx unless already
  - skip_if test -x /usr/sbin/nginx
  - yum install nginx
```

##### stop

Stops execution of the blebook. If called from a parent blebook, it will continue normally.

```bash
when 'test $USER = root' stop
```

##### stop_all

Stops execution of the blebook and all parent blebooks.

```bash
when 'test ! -f /etc/settings.cfg' stop_all
```

##### stop_if COMMAND ...

Runs the command and if it results in true, calls "stop".

```bash
stop_if test ! -f /etc/settings.cfg
stop_if not test -d /etc/nginx
```

##### stop_all_if COMMAND ...

Runs the command and if it results in true, calls "stop_all".

```bash
stop_all_if test ! -f /etc/settings.cfg
stop_all_if not test -d /etc/nginx
```

##### tags TAG ...

There is a special environment variable TAGS. It may contain tags separated by a space.
Then you can match these tags and skip actions.

Example:

```bash
@ Compiling redis
  - tags download compile
  - ./configure
  - make

@ Installing redis
  - tags install
  - make install

@ Starting redis service
  - service start redis
```

Now if you execute the blebook with no TAGS ...

 ```bash
  bashible redis.ble
 ```

=> all three blocks will happen

If you execute the blebook with TAGS ...

 ```bash
  TAGS="compile install" bashible redis.ble
 ```

=> again all three blocks will happen (in the third, there
   are no tags specified - it will run by default)

If you execute bashible with another TAGS ...

 ```bash
  TAGS="compile" bashible redis.ble
 ```

=> only the first and the third block will happen.


Moreover, if you put "tags" inside tasks blocks as above, it will affect only those blocks.
If you put "tags" before all tasks blocks, it will skip the whole blebook unless matched.


##### export_var NAME [COMMAND...] #####

Calls set_var to store the command's output (if any) and then exports the variable. Also preserves it over sudo (which normally cleans the environment).

```bash
export_var DOMAIN foobar.com
```

for more info see 'set_var'


##### unless 'EVALUATED STRING' COMMAND ...

see the "when" below (does the opposite)


##### when 'EVALUATED STRING' COMMAND ...

Runs the command if the evaluated string returns true ("when") or false ("unless").

```bash
when 'test -z "$ENVIRONMENT"' fail 'ENVIRONMENT variable is not set!'
when 'test ! -n "$ENVIRONMENT"' fail 'ENVIRONMENT variable is not set!'
unless '! test -z "$ENVIRONMENT"' fail 'ENVIRONMENT variable is not set!'
unless 'test ! -z "$ENVIRONMENT"' fail 'ENVIRONMENT variable is not set!'

@ Installing nginx unless already
  - when 'test -f /etc/nginx/nginx.conf' already
  - unless '! test -f /etc/nginx/nginx.conf' already
```

You can write more complex expressions,

```bash
when 'test -d /etc/nginx && test -d /var/www' skip
```


### Description of other functions

##### add_line LINE FILE

Add a line into the file (path) unless the line has been found already (somewhere in the file).

```bash
add_line 'UseDNS no;' /etc/ssh/sshd_config
```

##### append_line LINE FILE

Append a line into the file (path) unless the line has been found in the end already.

```bash
append_line 'UseDNS no;' /etc/ssh/sshd_config
```

##### comment_line_matching REGEXP FILE

Prefix matching line(s) by '#' in the file.

```bash
comment_line_matching 'UseDNS' /etc/ssh/sshd_config
```

##### install_gem NAME NAME ...

Install ruby gems unless already (the check is quick).

```bash
@ Installing application prerequisities
  - install_gem celluloid cuba sequel
```

##### only_user NAME

Fails if current user is not the one defined.

```bash
only_user root
```

##### rpm_is_installed NAME

Returns true if the rpm is installed.

```bash
unless 'rpm_is_installed nginx' fail 'nginx RPM is not installed'
```

##### prepend_line LINE FILE

Prepend the file with a line unless already.

```bash
prepend_line 'UseDNS no;' /etc/ssh/sshd_config
```

##### prepend_line LINE FILE

Prepend the file with a line unless already.

```bash
prepend_line 'UseDNS no;' /etc/ssh/sshd_config
```

##### remove_line_matching REGEX FILE

Remove line(s) matching regexp from the file.

```bash
remove_line_matching 'UseDNS' /etc/ssh/sshd_config
```

##### replace_line_matching REGEXP STRING FILE

Replace complete line where the matching regexp has been found.

```bash
replace_line_matching 'domain' 'domain=example.com' /etc/default/domain.cfg
```

##### replace_matching REGEXP STRING FILE

Replace matching regexps in the file with the string.

```bash
replace_matching 'enabled=0' 'enabled=1' /etc/default/foo.cfg
```

##### set_contents CONTENTS FILE

Write contents into the file, does nothing if already.

```bash
set_contents 'centos-local' /etc/default/hostname
```

##### set_contents_safe CONTENTS FILE

If the file is empty, write the contents into it. If the contents is already there, does nothing. Fails, if a different contents has already been there.

```bash
set_contents_safe 'centos-local' /etc/default/hostname
```

##### symlink SRC DEST

Creates a symbolic link from SRC to DEST unless already.

```bash
symlink '/usr/local/bin/redis' '/usr/bin/redis'
```

##### timeout SECS COMMAND ...

Runs the command but fails if it lasts more than SECS.

```bash
@ Start nginx service
  - service nginx start
  - timeout 20 wait_for_tcp 127.0.0.1:80 up
```

##### uncomment_line_matching REGEXP FILE

Removes '#' from the beginning of matching line(s) in the file.

```bash
uncomment_line_matching 'UseDNS' /etc/ssh/sshd_config
```

##### wait_for_tcp MATCH up|down

Waits for a listening tcp service to be up or down. Uses "netstat -ltn".

```bash
@ Stop nginx service
  - service nginx stop
  - timeout 20 wait_for_tcp 127.0.0.1:80 down
```

##### yum_install NAME NAME ...

Installs the specified software unless already (the check is quick).

```bash
@ Installing prerequisities
  - yum_install rsync vsftpd nginx
```


### FAQ

##### How to run a blebook a different user? (sudo)
If you want to run the whole blebook as another user, then either run bashible via sudo, 
```bash
sudo -u user1 bashible blebook.ble
``` 
or inside a blebook use "as" helper, 
```bash
 - as user1 command... 
```
or call a blebook from a blebook the same way,
```bash
as user1 call './another_blebook.ble'
```

note: The "as" helper does sudo in non-login shell mode, does preserve only PATH environment variable. 
      Depends on your sudoers file, if all used environment variables will be passed to the sudoed command. The environment cleanup is, for instance, a reason why bashible re-runs itself and doesn't just export all it's functions.
      Will be fixed in the future (see TODO.txt).

##### How to call a blebook from a blebook?

```bash
call '/another/path/to/blebook.ble'
```
The working directory of the called blebook will be set to the same directory as the blebook's resides in, but won't affect later commands in the calling blebook.

##### How to skip tasks which have been already done?

either using an universal "when" helper,

```bash
@ Install nginx unless already
   - when 'test -f /usr/bin/nginx' already
   - yum install -y nginx
```
or the same using "already_if" helper

```bash
@ Install nginx unless already
   - already_if test -f /usr/bin/nginx
   - yum install -y nginx
```

##### How to write more complex if expressions?

```bash
@ Install nginx unless its executable and configuration file exists
   - when 'test -f /usr/bin/nginx && test -f /etc/nginx/nginx.conf' already
   - yum install -y nginx
```
(the first argument of the "when" helper is evaluated as a bash)

##### How to check if a blebook is executed under a certain user?

```bash
unless 'test $USER = user1' fail "only user1 can run this blebook"
```

##### How to use more complicated bash code as a task?
```bash
- already_if eval 'foo bar baz'
```

##### How to extend bashible with my own functions (helpers)?
```bash
import /etc/my/functions.sh
```
note: using "source" is not sufficient; sourced functions are not populated into child processes (when another blebooks are called from the parent one), also are not usable after sudo which usually cleans environment variables (so even if functions have been exported).

##### How to set a variable depending on a task beeing executed?

use "export",

```bash
@ Installing nginx unless already
  - already_if test -d /etc/nginx
  - yum install nginx
  - set_var NGINX_INSTALLED echo 1

@ Configuring nginx if freshly installed
  - skip_if test $NGINX_INSTALLED = 1
  - cp -a /defaults/nginx/* /etc/nginx/
```
