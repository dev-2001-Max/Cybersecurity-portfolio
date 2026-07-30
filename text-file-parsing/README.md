# Automating Allow List Updates with Python

## Objective
Wrote a Python algorithm to automate maintaining an IP address allow list — reading
an existing allow-list file, removing IP addresses flagged for revocation, and writing
the updated list back to the file.

## Scenario
An organization controls access to restricted content using an `allow_list.txt` file
containing approved IP addresses. When IP addresses need their access revoked
(tracked in a separate `remove_list`), the file needs to be updated to exclude them —
automating this avoids manual, error-prone editing of the file.

## Algorithm

**1. Open and read the allow list file**
```python
import_file = "allow_list.txt"

with open(import_file, "r") as file:
    ip_addresses = file.read()
```
Used a `with` statement to open the file in read mode (`"r"`), which also ensures the
file is automatically closed afterward. `.read()` converts the file's contents into a
single string.

**2. Convert the string into a list**
```python
ip_addresses = ip_addresses.split()
```
`.split()` breaks the string into a list of individual IP addresses (splitting on
whitespace by default), making each entry easy to check and remove individually.

**3. Iterate through the remove list and remove matches**
```python
for element in remove_list:
    if element in ip_addresses:
        ip_addresses.remove(element)
```
Looped through each IP in `remove_list`, checking whether it existed in
`ip_addresses` before removing it — this conditional check avoids a runtime error
that `.remove()` would raise if the element wasn't present.

**4. Convert the list back into a string and write it to the file**
```python
ip_addresses = "\n".join(ip_addresses)

with open(import_file, "w") as file:
    file.write(ip_addresses)
```
`.join()` reassembled the list into a newline-separated string. Opening the file in
write mode (`"w"`) and calling `.write()` overwrote the original file with the revised
allow list, revoking access for every removed IP address.

## Summary
Built a Python algorithm that automates removing revoked IP addresses from an
access-control allow list — reading the file, converting it to a list for safe iteration
and removal, then writing the updated list back to disk. This demonstrates practical
scripting for access-control maintenance, a task otherwise prone to manual error.

## Skills Demonstrated
Python file I/O (`open`, `read`, `write`), string/list manipulation (`split`, `join`),
conditional logic and iteration, automation for access control management