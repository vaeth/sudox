# sudox

(C) Martin Väth (martin at mvath.de).
This project is under the BSD license 2.0 (“3-clause BSD license”).
SPDX-License-Identifier: BSD-3-Clause

sudox is a POSIX shell script which acts as a wrapper for
`sudo -H` [`-s`] which can pass X cookies.

It is possible to create temporary untrusted X permissions.
Moreover, wayland is supported via acls, variable passing, and a wrapper.
Also some support for tty handling (e.g. screen and tmux) is provided.

For more details, see the output of `sudox -h`

If you want to change the user for the remainder of the whole shell session
and are somewhat paranoic to do this by accident from a runnnig
screen or tmux session, you can use `sudoxe` in place of `sudox`:
This is essentially `exec sudox`, but checks your environment first.
Be aware that when using `sudoxe` (or `exec sudox`) and `sudo` asks for a
password, a wrong password will terminate your session anyway.
(Please let me know if you know a trick how to avoid this problem...)


## Installation

For installation, copy `bin/*` to your `$PATH` and add code like
```
if SOME_VARIABLE=`sudoxe 2>/dev/null`
then	eval "$SOME_VARIABLE"
else	echo 'sudoxe not found' >&2
fi
```
to your shell startup file. Alternatively, you can also simply use
```
. sudoxe
```
but the latter has the disadvantage that your shell will die if sudoxe
cannot be found.

The above code will define the shell functions `sudox()` and `sudoxe()`
(`sudox()` calls sudox with a secure `PATH` setting). Alternatively, add these
functions or modifications thereof directly to your shell startup file.

To obtain support for zsh completion, you can copy `zsh/*` to a directory
of zsh's `$fpath`.

You need `push.sh` from https://github.com/vaeth/push (v2.0 or newer)
in your `$PATH` as well.

For wayfire support, copy also
```
usr/share/wayland-sessions/wayfire-auth.desktop
```
to the wayland session directory of the display-manager
(presumably `/usr/share/wayland-sessions`), see the next section.

For Gentoo, there is an ebuild in the mv overlay (available by layman).


## Xwayland

In order for sudox to take effect on `Xwayland`, `Xwayland` must be run with
the options
```
Xwayland $DISPLAY -auth $XAUTHORITY ...
```
(and the `$XAUTHORITY` file must have been created).
Depending on the compositor and how it is initialized, this might not be true,
automatically.

In particular, for wlroots based compositors like sway or wayfire, this is
usually not the case.

A workaround for wayfire (and similarly for other wlroots based compositors)
may be to use the provided `wayfire-auth.desktop`. This works as follows:

Instead of directly calling `wayfire`, this file calls `wayfire-auth`
which is a script ending with
```
WLR_XWAYLAND=`command -v xwayland.auth`
export WLR_XWAYLAND
exec wayfire "$@"
```
The variable `WLR_XWAYLAND` means for wlroot that `xwayland.auth` is used
instead of `Xwayland`. And `xwayland.auth` in turn is a wrapper for `Xwayland`
which fills `$HOME/.Xauthority` as a new `$XAUTHORITY` file and calls
`Xwayland` in the way described above.


## Security notes

X cookies should be kept secret, because every user who knows them can access
the running X session. The main purpose of __sudox__ is to pass such X cookies
to the freshly started __sudo__ process.
Below are described all 4 methods (and a method avoiding X cookie passing)
which can be used by __sudox__ for this.
Each of these methods has certain restrictions or security implications;
therefore you can choose by options.
The environment variable `SUDOX_OPT` can be used to set options which
preselect some of these methods.
Since __sudox-8.0.0__, the most secure option is the default; therefore,
`SUDOX_OPT` should only be set if no other option can be used.
`SUDOX_OPT` thus is a potential security issue and it will perhaps be
ignored in some future relase of __sudox__, so try to not rely on it.

### 1. Best method: "Environment variable" method

Since __sudox-8.0.0__, this method is the default with variable name `DISPLAY`.
This method requires that `sudo` is configured to keep the variable (`DISPLAY`)
in the environment (see the comments in the provided `sudoers.d`).
To specify a different variable name, use the sudox option `-vVAR`
If the __sudo__ configuration cannot be modified and is not known and the
default `DISPLAY` does not work, try one of the following values for `VAR`
first:

- `COLORS`
- `HOSTNAME`
- `LS_COLORS`
- `PS1`
- `PS2`
- `XAUTHORITY`
- `XAUTHORIZATION`

Less optimal (though perhaps working) are __sudo__ checked variables like
these:

- `COLORTERM`
- `LANG`
- `LANGUAGE`
- `LC_*`
- `LINGUAS`
- `TERM`
- `TZ`

Every name matching the regular expression `^[A-Z_a-z][A-Za-z0-9_]*`
can be used;
excluded are only those names which match `^sx_[a-z]` or which influence shell
execution like

- `PATH`
- `IFS`
- `BASH_ENV`
- `ENV`
- `SHELL_OPTS`

The variable will only be used temporarily by __sudox__ for the transfer; the
original content will be restored after this temporary usage (__sudox__ will
even unset the variable if it was not in the original environment).

__Security Implication__: Do not use this method if your system is such that
the (initial) process environment can be read by other users (on linux, check
e.g. that `/proc/*/environment` is only user readable).
Otherwise, your X cookies are leaked this way!

### 2. Second best method: File descriptors

If the method above is not possible (e.g. because __sudo__ is configured to not
pass _any_ variable and you cannot change the configuration), __sudox__ can use
an open file descriptor to pass X cookies. This method is secure, but only
suboptimal, because `sudox` still needs to keep a process running to fill
the descriptor and cannot terminate until the `sudo` program returns:
The `sudo` process becomes a child of the `sudox` process (with possible
security implications about sending signals to an unprivileged process).
Use the `sudox` option `-F#` or `-F#,#a` to use this method with file
descriptor `#` as channel for X cookie passing and an auxiliary file
descriptor `#a`.
As `#` you can either use some unused descriptor (at least `3`) or the
standard input `0`. Depending on your shell, `#` and `#a` must be at most `9`.

- If `#` is at least `3` then __sudo__ must be configured to not close this
  file descriptor, e.g. by allowing `closefrom_override`
  (see the provided `sudoers.d`) and passing a corresponding `-C` option.
  To use this mode, also `#a` is needed which should be at least `3` and
  not be used for another purpose (but which can be closed by `sudo`).
  If `#a` is not specified or empty, then (`#` + 1) is chosen.
  A typical example usage of this option is `-F3` (which is the same as
  `-F3,4`).

  Unless you configured __sudo__ to use `closefrom=4` (or higher), you have to
  combine this example with the option `-C4` (which works only if sudo is
  configured to allow `closefrom_override`):

  `sudox -F3 -C4 user command`

- If `#` is `0` then no special __sudo__ configuration is needed. However, the
  executed command will have its standard input (`0`) closed. If you specify
  `#a` as `0` then the `cat` command will be used to pipe standard input to the
  standard input of the executed command, but the latter will break most
  interactive programs and also has security implication (`cat` runs as the
  calling user). Therefore, it is recommended to use `-F0` only with programs
  which do not expect standard input. In this case, it is recommended to
  combine this with `-T`.
  To first transfer X cookies and then to call an interactive program you can
  instead call sudox twice:

  `sudox -TF0 user true && sudox -X user [command ...]`

### 3. Not very secure method: FIFO

If none of the above methods can be used (e.g. because __sudo__ cannot be
configured and does not pass any variable or file descriptor) and you
do not want to use the file descriptor `0` workaround by calling __sudox__
twice, __sudox__ can use a FIFO in a randomly generated temporary directory
(whose name is transferred over a command line) to transfer X cookies.
Since the FIFO has to be world readable, this method is insecure, because
an attacker can read the X cookies from this FIFO until the freshly started
`sudo` process reads it. Although __sudox__ will recognize this and stop
with an error about a Troyan, the attacker might misuse the X cookies already
and hide the display of this information.
However, the timeframe for the attacker to read the FIFO is only from the
call of `sudox` until the freshly started `sudo` process reads it.

Another disadvantage of the method is the same as that of 2:
__sudox__ needs to keep a process running to fill the FIFO and cannot terminate
until the `sudo` program returns: The `sudo` process bcomes a child of `sudox`.
To select this method with `sudox`, use the option `-F-` (which is the same
option as for 2. but with `#` being `-`).

### 4. Insecure method: Command line

As a final fallback, the X cookies can be passed by the command line.
On most systems, the command line of a process can be accessed by every user,
so this is usually an insecure way of passing X cookies.
To select this method with sudox, use the option `-v-` (which is the same
option as for the "Environment Variable" method but with the special
variable name `-`) (and do not use the `-F` option).

### root mode alternative

If the destination user has the permission to access the calling user's
`XAUTHORITY` file (default: `~/.Xauthority`), then `XAUTHORITY` can be set
to that file. This has the implication that also all modifications of
permissions go to that file. In particular, this method cannot be used
to generate untrusted permissions for the destination user.
This method is used by `sudox` automatically if the destination user is root
and no untrusted permissions are requested. To override this default, use
the options `-R` (to force root mode) or `-N` (to force non-root mode).
