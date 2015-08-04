#### reset_base_dir PATH

Sets a base directory to the original (the same directory as where the script.ble resides in).

Example:

```bash

base_dir '/etc'

@ Working in /etc
  - cp passwd passwd.bak
  - cp shadow shadow.bak

@ Still working in /etc
  - cp group group.bak

reset_base_dir

@ Now working in the original directory
  - ls

```

##### See also

[base_dir](base_dir.md)

