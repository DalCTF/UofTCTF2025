# Racing 2

## Analysis

This problems builds on the problem [Racing](/writeups/UofTCTF2025_Misc_Racing). Once again, we are given the [source code](./chal.c) for a `C` program and a machine to connect to via `SSH`. The machine contained an executable of the source code provided with the `setuid` flag enabled inside the `/challenge` folder. It also contained a file `flag.txt` on the system's root.

## Solution

Instead of having a program that reads a file - like in the previous problem - this time the program can **write** to a file on the path `/home/user/permitted`. We've seen on the previous problem that we can create a _symlink_ on that location and, essentially, make the program access another file on the system. Since the program has the `setuid` bit enabled, the program will run as the `root` user and, therefore, is able to override any file on the system that `root` would normally have permission to. 

However, the program uses the `access` function to check if the file is accessible by the current user before attempting to write. While the write operation would be successful because the program runs as the super user, the `access` function will provide us access levels for the current session's user, aptly named `user`, in this case. If it is a protected file, the program will fail the `access` check and end.

So there are two things at play here: the moment it checks if the user has proper `access` to the file, and another where it writes over the contents of that file. This is a case of [TOCTOU](https://en.wikipedia.org/wiki/Time-of-check_to_time-of-use), that is, _time of check, time of update_. In a nutshell, this means that there is a moment between the check (our `access` function call) and the time of update (writing to the file) where the state of the system can change.

In our case here, the moment the program asks for the content to be written on the file, it has already performed the check, but not the update. That means that, on this moment, we can perform another action that changes the `/home/user/permitted` file into something different, and then resume the execution of the program, so that it operates on the new file. To leave the context of the program and return to our shell session, we can press `[Ctrl-Z]`.

Therefore, in order to write to a different file we would do something like:

```bash
$ touch /home/user/permitted
$ /challenge/chal

Enter text to write: 
[Ctrl-Z] # This takes us back to our shell session

$ rm /home/user/permitted
$ ln -s <FILE TO WRITE> /home/user/permitted
$ fg # This takes us back to the program
<CONTENT TO WRITE>
```

This will write `<CONTENT TO WRITE>` to the file `<FILE TO WRITE>` using our super user permissions.

We still do not know exactly which file we can change that will make us able to access the `/flag.txt` file. While there are many files we could change on the system, the file `/etc/passwd` controls the user ids (`UID`) and group ids (`GID`) of the users on the system. If we write to this file, we can specify the our current user, "user", is, in fact, the one with `UID = 0`, which is the one that owns the `/flag.txt` file.

Therefore, we can run the following:

```bash
$ touch /home/user/permitted
$ /challenge/chal

Enter text to write: 
[Ctrl-Z]

$ rm /home/user/permitted
$ ln -s /etc/passwd /home/user/permitted
$ fg
user::0:0:admin:/root/root:/bin/bash
```

If you look at the contents of `/etc/passwd`, you will see that it has in fact changed, but if you try to read `/flag.txt`, you will still get a message `cat: /flag.txt: Permission denied`. This is because the current `UID` of the terminal session is still set to the old value, not the one we updated on `/etc/passwd`. If we try to run `bash`, we will get a new session, but it will retain the old `UID`. In order to make it reload the `UID` value, we need to use:

```bash
$ su user
```

This will give us a new session where our `UID` is reloaded from `/etc/passwd`, that is, it is now `UID = 0`. Finally, we can read `/flag.txt`:

```bash
$ cat /flag.txt
uoftctf{f1nn_mcm155113_15_my_f4v0r173_ch4r4c73r}
```