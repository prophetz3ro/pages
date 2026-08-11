---
layout: page
title: Linux permissions
permalink: /linux_perm/
---

# Linux Permissions — Important Edge Cases

Linux permissions are divided into:

- **owner**
- **group**
- **others**

Each class can have:

- `r` — read
- `w` — write
- `x` — execute

The meaning of these bits differs between regular files and directories.

---

## Directory permissions

For a directory:

- `r` — list filenames inside the directory
- `w` — create, delete, or rename directory entries
- `x` — traverse the directory and access entries by name

### Read without execute

```bash
chmod 400 dir
```

This may allow:

```bash
ls dir
```

But it does not allow:

```bash
cd dir
cat dir/file
stat dir/file
```

Without `x`, the system cannot access entries inside the directory.

`ls -l dir` may show permission errors or question marks because it can read filenames but cannot retrieve their metadata.

### Execute without read

```bash
chmod 100 dir
```

You cannot list the directory:

```bash
ls dir
```

But if you know the exact filename, you may access it:

```bash
cat dir/file
```

The file itself must also allow reading.

### Useful combinations

```text
r--  list names, but cannot access entries
--x  access known entries, but cannot list names
r-x  list and access entries
-wx  create, delete, and rename entries without listing
rwx  full directory access
```

---

## Deleting files

Deleting a file is mainly an operation on its parent directory.

```bash
chmod 000 dir/file
chmod 777 dir
rm dir/file
```

The file can still be deleted because the user has `w+x` on the parent directory.

The file's own permissions control access to its contents, but normally do not control deletion of its directory entry.

---

## `umask`

`umask` removes permission bits when files and directories are created.

Typical requested permissions are:

```text
regular file: 666
directory:    777
```

Regular files do not receive execute bits by default.

Conceptually:

```text
result = requested permissions AND NOT umask
```

### `umask 022`

```bash
umask 022
touch file
mkdir dir
```

Results:

```text
file: 644
dir:  755
```

### `umask 077`

```bash
umask 077
touch file
mkdir dir
```

Results:

```text
file: 600
dir:  700
```

`umask` specifies which bits should be disabled, not the final permissions.

---

## Special permission bits

```text
4xxx — SUID
2xxx — SGID
1xxx — sticky bit
```

Examples:

```bash
chmod 4755 program
chmod 2775 shared
chmod 1777 public
```

---

## SGID on directories

SGID on a directory causes newly created files and subdirectories to inherit the directory's group.

```bash
mkdir shared
chgrp somegroup shared
chmod 2775 shared
```

Check it:

```bash
ls -ld shared
```

Example:

```text
drwxrwsr-x
```

The `s` in the group execute position indicates SGID.

Test:

```bash
sudo -u user1 touch shared/file1
ls -l shared/file1
```

The new file should inherit `somegroup` instead of the user's primary group.

---

## Sticky bit

The sticky bit is mainly used on shared writable directories such as `/tmp`.

```bash
mkdir public
chmod 1777 public
```

Check it:

```bash
ls -ld public
```

Expected:

```text
drwxrwxrwt
```

The final `t` indicates the sticky bit.

Without sticky bit, anyone with `w+x` on a directory can normally delete files belonging to other users.

With sticky bit, a file can normally be deleted or renamed only by:

- the file owner
- the directory owner
- root

Test:

```bash
sudo -u user1 touch public/file1
sudo -u user2 rm public/file1
```

The second command should fail:

```text
Operation not permitted
```

### SGID versus sticky bit

SGID:

```text
drwxrwsrwx
      ^
      s
```

Sticky bit:

```text
drwxrwxrwt
         ^
         t
```

Both can be combined:

```bash
chmod 3777 shared
```

```text
drwxrwsrwt
```

---

## SUID

SUID is normally used on executable files.

```bash
chmod u+s program
```

Numeric form:

```bash
chmod 4755 program
```

Check it:

```bash
ls -l program
```

Example:

```text
-rwsr-xr-x
```

The `s` in the owner execute position indicates SUID.

When the program runs:

```text
real UID      = user who started the process
effective UID = owner of the executable
```

Example:

```bash
sudo chown root:root program
sudo chmod 4755 program
```

A normal user running this executable receives the permissions of the file owner for operations performed by that program.

SUID does not automatically provide a shell. The program decides what privileged operations are available.

SUID on interpreted scripts is normally ignored by Linux. It should be tested with a compiled executable.

### Lowercase `s` and uppercase `S`

```text
-rwsr-xr-x
```

Lowercase `s` means SUID and owner execute are both set.

```text
-rwSr-xr-x
```

Uppercase `S` means SUID is set, but owner execute is missing.

---

## ACLs

ACLs allow permissions to be assigned to specific users or groups outside the normal owner/group/others model.

```bash
touch secret
chmod 600 secret
setfacl -m u:user2:r secret
```

Now `user2` can read the file:

```bash
sudo -u user2 cat secret
```

Compare:

```bash
ls -l secret
getfacl secret
```

`ls -l` may show:

```text
-rw-r-----+
```

The `+` indicates that ACL entries exist.

`getfacl` shows the actual rules:

```text
user::rw-
user:user2:r--
group::---
mask::r--
other::---
```

The ACL mask limits the effective permissions of named users and groups.

---

## Exercises

### 1. Directory permissions

```bash
mkdir -p lab/dir
echo secret > lab/dir/file
cd lab
```

Test:

```bash
chmod 400 dir
ls dir
ls -l dir
cat dir/file
cd dir
```

Then:

```bash
chmod 100 dir
ls dir
cat dir/file
```

Then test:

```bash
chmod 500 dir
chmod 700 dir
```

Observe which operations require `r` and which require `x`.

### 2. Deleting a file with mode `000`

```bash
mkdir delete-test
touch delete-test/file
chmod 000 delete-test/file
chmod 777 delete-test
rm delete-test/file
```

Observe that deletion depends on the parent directory.

### 3. `umask`

```bash
umask 022
touch file1
mkdir dir1
ls -ld file1 dir1
```

Expected:

```text
file1: 644
dir1:  755
```

Then:

```bash
umask 077
touch file2
mkdir dir2
ls -ld file2 dir2
```

Expected:

```text
file2: 600
dir2:  700
```

### 4. SGID directory

```bash
sudo mkdir /tmp/shared-test
sudo chown root:somegroup /tmp/shared-test
sudo chmod 2775 /tmp/shared-test
```

Create files as different users:

```bash
sudo -u user1 touch /tmp/shared-test/file1
sudo -u user2 touch /tmp/shared-test/file2
ls -l /tmp/shared-test
```

Both files should inherit `somegroup`.

### 5. Sticky bit

```bash
sudo mkdir /tmp/sticky-test
sudo chmod 1777 /tmp/sticky-test
sudo -u user1 touch /tmp/sticky-test/file1
sudo -u user2 rm /tmp/sticky-test/file1
```

The last command should fail with:

```text
Operation not permitted
```

### 6. ACL

```bash
touch secret
chmod 600 secret
echo "classified" > secret
setfacl -m u:user2:r secret
sudo -u user2 cat secret
```

Compare:

```bash
ls -l secret
getfacl secret
```