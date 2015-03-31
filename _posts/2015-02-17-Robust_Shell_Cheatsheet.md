---
style: posts
---

#Robust Bash Shell Scripting
 Cheatsheet based on David Pashley's blog
 [post](http://www.davidpashley.com/articles/writing-robust-shell-scripts/)

## Use the 'set' builtin
1) Prevent unset variables: `set -u` or `set -o nounset`
1) Avoid race conditions between file checking/creation: `set -C` or `set -o noclobber`
1) Exit script if something fails: `set -e` or `set -o errexit`
  * (!) You cannot check the exit status with `$?` with this on
  ** Alternatives
    *** `command || {echo "Failed"; exit 1;}`
    *** `if !command; then echo "Failed"; exit 1; fi`
  * To skip checking of a line: `command || true`
  * To skip checking of a section:
  ```shell
  set +e
  command
  ...
  set -e
  ```
1) Allow traceback (using `trap`) with `set -E` or `set -o errtrace`

## Program defensively
1) `rm -f path` over `rm path` - won't fail if path d.n.e.
1) `mkdir -p path` over `mkdir path` - won't fail if parent dir d.n.e.
1) `"$filename"` over `$filename` - won't fail if filename has spaces
1) `"$@"` over `$@` - will expand correctly if input(s) have spaces
1) `find -print0 | xargs -0 command` over `find | xargs command` - will expand filenames with spaces correctly

#### Examples
Expanding spaces

```shell
#!/bin/sh
foo () {
  for i in $@; do
    printf "%s\n" "$i"
  done
}

bar () {
  for i in "$@"; do
    printf "%s\n" "$i"
  done
}

foo baz "qux quz"
# Output:
# baz
# qux
# quz
bar baz "qux quz"
# Output:
# baz
# qux quz  <-- correct!
```
See
[Example 9-7](http://tldp.org/LDP/abs/html/internalvariables.html#INCOMPAT)
for a more in-depth example

## Use 'trap' to prevent inconsistent states
1) Use `trap ACTION SIGNAL [SIGNAL...]` to guarantee lock files get
removed, clean up code is run, etc.
  * Signals of interest:
  ** `INT` - interrupt via "Ctrl-C",
  ** `TERM` - user terminated via
  "kill"),
  ** `EXIT` - script exits either by reaching end or failing command (with `set -e`)

#### Example
```#!/bin/sh
...
if [[ ! -e "$LOCKFILE" ]]; then
  touch "$LOCKFILE"
  critical_section  # <--- if this fails, the lock file will persist!
  rm "$LOCKFILE"
else
  echo "Critical operations running"
fi
```
```#!/bin/sh
...
if [[ ! -e "$LOCKFILE" ]]; then
  trap "rm -f $LOCKFILE; exit" INT TERM EXIT
  touch "$LOCKFILE"
  critical_section  # <--- if this fails, trap will delete the lock file!
  rm "$LOCKFILE"
  trap - INT TERM EXIT  # Turn off trap!!! (i.e. reset the signals to their values when shell started)
else
  echo "Critical operations running"
fi
```
Avoid race condition with `LOCKFILE`:
```#!/bin/sh
...
if ( set -o noclobber; echo "$$" > "$LOCKFILE") 2>/dev/null; then
  trap "rm -f $LOCKFILE; exit $?" INT TERM EXIT
  touch "$LOCKFILE"
  critical_section  # <--- if this fails, trap will delete the lock file!
  rm "$LOCKFILE"
  trap - INT TERM EXIT  # Turn off trap!!! (i.e. reset the signals to their values when shell started)
else
  echo "Failed to acquire lock file"
  echo "Held by $(cat $LOCKFILE)"
fi
```

## Race conditions
1) Use "rollback" functions to undo actions if `trap` is caught:
```shell
rollback () {
  rm -rf "${NEW_DIR}"
}

trap rollback INT TERM EXIT
cp -R "${OLD_DIR}" "${NEW_DIR}"
chown -R ${USER} "${NEW_DIR}"
trap - INT TERM EXIT  # <--- turn off or trap will call rollback on
script exit!!!
exit 0
```

## Be atomic
Change a file-by-file modification script to copy the files to a
temporary location, modify the temporary location, then move them to
the location of the originals.  Though you have to use a lot of disk
space, it is very safe and easily recovered from.
