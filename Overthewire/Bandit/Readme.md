# Bandit - All levels


* **Level - 0**

  SSH as **bandit0** with `ssh -p 2220 bandit0@bandit.labs.overthewire.org` giving password **bandit0**.


* **Level - 0**

  After logging in as **bandit0** do 
  ```bash
    bandit0@bandit:~$ ls
    readme
    bandit0@bandit:~$ cat readme
    boJ9jbbUNNfktd78OOpsqOltutMc3MY1
    ```
    The hash is the password for **bandit**.
    
 * **Level - 1**
 
    After loggin in as **bandit1** and running `ls` we see a file named **-**. The problem is you can't directly `cat` that 
    because **-** is another name for **stdin** in **Linux** , and therefore `cat -` will just tell **cat** to wait for args 
    from **stdin**.
    
    Bypass this giving the full path with `./-`.
    
    ```bash
    cat ./-
    CV1DtqXWVFXTvM2F0k09SHz0YwRINYA9
    ```
    
    
 * **Level - 2**
    
    After running `ls` you will see a file that has spaces in its name. There are two ways to access such files in **bash**
    ```bash
    bandit2@bandit:~$ ls
    spaces in this filename
    bandit2@bandit:~$ cat "spaces in this filename"
    UmHadQclWmgdLOKQ3YNgjWxGoRMb5luK
    bandit2@bandit:~$ cat spaces\ in\ this\ filename
    UmHadQclWmgdLOKQ3YNgjWxGoRMb5luK```
      
 * **Level - 3**

    Use `-a` parameter for `ls` to show **all** files. Read more with `man ls`.
    ```bash
    bandit3@bandit:~$ ls
    inhere
    bandit3@bandit:~$ cd inhere
    bandit3@bandit:~/inhere$ ls
    bandit3@bandit:~/inhere$ ls -la
    total 12
    drwxr-xr-x 2 root    root    4096 Oct 16  2018 .
    drwxr-xr-x 3 root    root    4096 Oct 16  2018 ..
    -rw-r----- 1 bandit4 bandit3   33 Oct 16  2018 .hidden
    bandit3@bandit:~/inhere$ cat .hidden
    pIwrPrtPN36QITSp3EQaw936yaFoFgAB
    ```
    

 * **Level - 4**
 
 
    After running `ls` into **inhere**, running the `file` command we find only one file containing ASCII data.
    <!-- bandit4 -->
    
* **Level - 5**    

     The page informs us that
     > The password for the next level is stored in a file somewhere under the inhere directory and has all of the following             properties: - human-readable - 1033 bytes in size - not executable
    
    You can just use the first bit of info and run `du -a -b | grep 1033` or all and run `find ./ -type f -size 1033c ! -executable`
    
    ```bash
    bandit5@bandit:~/inhere$ cat ./maybehere07/.file2
    DXjZPULLxYr17uwoI01bNLQbtFemEgo```
    
* **Level - 6** 
