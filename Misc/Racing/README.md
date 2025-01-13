# Racing

## Analysis

The problem provides us the [source code](./chal.c) of a `C` program and a machine to be connected via `SSH`. The machine contained an executable of the source code provided with the `setuid` flag enabled inside the `/challenge` folder. It also contained a file `flag.txt` on the system's root.

## Solution

By looking at the source code, we will notice that it uses `setuid(0)` to run as the `root` user. This is possible because of the `setuid` bit we mentioned previously.

Next, it checks if the program has the proper permissions to access the file on the path `/home/user/permitted`. If it does, it prompts the user for a file name. It will not allow the file name to contain the word `flag`. If the name is empty, it will default to the path `/home/user/permitted`. Once the path is defined, the file is read and its contents are printed to console.

Trying to read the file `/flag.txt` is not going to work because that path contains "flag", which is stopped by the program. We cannot rename the file because we don't have the proper permissions for it. However, we can pass to the program the path to a **symbolic link** (_symlink_) to the `/flag.txt`. A symbolic link will point to the same file, but will live in a different location and have a different name.

Therefore, we create a symlink by doing:

```bash
$ ln -s /flag.txt /home/user/permitted
```

This will create a _symlink_ to the flag file on `/home/user/permitted`. Since the program has access to it and the file does not contain "flag" in its name, it should be read the contents of the link. Being a link, that means that the content read will be that of `/flag.txt`.

Therefore, we can execute the program and enter either "/home/user/permitted" or simply leave the input blank (which will default to the same path), and we will get the flag:

```bash
$ ln -s /flag.txt /home/user/permitted
$ /challenge/chal

Enter file to read: 
uoftctf{r4c3_c0nd1t10n5_4r3_c00l}
```