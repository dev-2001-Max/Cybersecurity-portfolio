# Linux File Permissions Management

## Objective
Reviewed and updated file and directory permissions within a Linux research team's
project directory to align access levels with organizational security requirements.

## Approach

### 1. Check current permissions
Used `ls -la` to list all files/directories with their full permission strings, including
hidden files:
```
ls -la
```
This revealed permission strings such as `-rw-rw-rw-` (project_k.txt), `-rw-r-----`
(project_m.txt), and `drwx--x---` (drafts directory) — each representing read/write/
execute access for user, group, and other.

### 2. Interpret the permission string
Broke down the standard 10-character Linux permission string:
- Character 1: file type (`d` = directory, `-` = file)
- Characters 2–4: user (owner) permissions
- Characters 5–7: group permissions
- Characters 8–10: permissions for other (everyone else)

### 3. Remove excess write access
Identified that `project_k.txt` granted write access to "other," which violated the
requirement that no external users should be able to modify files:
```
chmod o-w project_k.txt
```

### 4. Adjust permissions on a hidden/archived file
For an archived file (`.project_x.txt`), removed write access from both user and group
while adding read access for the group, since the file was no longer being edited:
```
chmod u-w,g-w,g+r .project_x.txt
```

### 5. Restrict directory access
Removed group execute permissions on the `drafts` directory so only the designated
owner (`researcher2`) retained access:
```
chmod g-x drafts
```

## Summary
Used `ls -la` to audit existing permissions, then applied targeted `chmod` changes to
enforce least-privilege access — removing unnecessary write and execute permissions
from files and a directory to match the organization's required access-control policy.

## Skills Demonstrated
Linux command-line administration, file permission auditing (`ls -la`), permission
modification (`chmod`), least-privilege enforcement