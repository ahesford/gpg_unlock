# gpg_unlock

This is a simple Python 3 script that reads, from `stdin`, the first line of
input as a passphrase that is then passed to `gpg-connect-agent` to unlock one
or more keys. By design, `gpg-connect-agent` will start `gpg-agent` if
necessary. This script is intended primarily to be called from the
`pam_exec.so` module, but may be used from a shell. Any trailing carriage
return or newline is stripped from the password.

### Usage

```
	gpg_unlock [KEYLIST ...]
```

Read the given `KEYLIST` files for lists of GPG keygrips and, if at least one
keygrip is found, read a passphrase from stdin. If and only if no `KEYLIST`
files are specified, `$HOME/.gnupg/autounlock` will be checked by default.

`KEYLIST` files may be commented; comments begin with a `#` (either at the
start of the line or after some whitespace) and continue to the end of the line
on which they appear. Lines that are empty or consist of all whitespace after
comments are removed will be ignored. Any files that cannot be read will cause
an error message to be emitted to `stderr`, but processing of other files will
continue.

For the purposes of unlocking, a keygrip is any whole word in a `KEYLIST` file.
More than one keygrip may be specified per file, with one or more per line.
Valid keygrips may be identified by running
```
	gpg2 --list-secret-keys --with-keygrip
```
(For some distributions, `gpg` may be needed in place of `gpg2`.)

No attempt is made to validate keygrips before `gpg-connect-agent` is invoked
to unlock keys; `gpg-agent` should simply refuse to accept invalid keys.

`gpg-unlock` may be invoked from PAM by adding
```
auth optional pam_exec.so expose_authtok /path/to/gpg_unlock
```
in the relevant PAM service files. The line must appear _AFTER_ the module
responsible for authenticating users (commonly `pam_unix.so`) for the
authentication token to be passed on `stdin` via the `expose_authtok`
mechanism. On Void Linux, adding the line to `/etc/pam.d/system-local-login`
after the line
```
auth include system-login
```
will perform the unlock only on local console logins.

To use `gpg_unlock` outside of PAM, invoke
```
stty -echo && gpg_unlock
```
from a shell and enter a passphrase, followed by `<Enter>`. 

`gpg_unlock` will forward any `stdout` or `stderr` output from
`gpg-connect-agent` to its own `stdout` and `stderr`, respectively.
`gpg_unlock` also prints some error conditions (not all of which are fatal) to
`stderr`. Generally, `gpg-connect-agent` writes command responses to `stdout`;
the `pam_exec` module will ignore `stdout` by default. There is currently no
way to suppress or reconfigure output in `gpg_unlock` except to edit the
script.


### Privileges

When invoked from a PAM module, `gpg_unlock` may be run as the superuser rather
than as the user being authenticated. When `gpg_unlock` finds a value in
`PAM_USER` (which is set by `pam_exec.so` to the user being authenticated), it
will immediately attempt to adopt the UID and primary GID of `$PAM_USER` and
relinquish any other group privileges. If these attemps fail, `gpg_unlock` will
exit with a unit return code before attempting further action.

As a special case, if `$PAM_USER` has the same value as `os.getuid()` prior to
dropping privileges, `gpg_unlock` will not attempt to alter group privileges.
