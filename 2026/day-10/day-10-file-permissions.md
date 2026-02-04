# Day 10 Challenge
---
## Files Created
- demo  
- devops.txt 
- gurudeen  
- notes.txt 
- project  
- script.sh  
- trainwithsubham
---
---
## Permission Changes
# [before]
total 8
- drwxr-xr-x. 5 root root 52 Jan 29 23:20 `demo`
- -r--r--r--. 1 root root  0 Feb  4 21:49 `devops.txt`
- drwxr-xr-x. 3 root root 18 Jan 27 16:15 `gurudeen`
- -rw-r--r--. 1 root root 22 Feb  4 21:50 `notes.txt`
- -rwxr-xr-x. 1 root root 21 Feb  4 21:51 `script.sh`
- drwxr-xr-x. 3 root root 28 Jan 29 21:59 `trainwithsubham`

# [after for each file]
total 12
- drwxr-xr-x. 5 root root 52 Jan 29 23:20 `demo`
- -r--r--r--. 1 root root  7 Feb  4 22:21 `devops.txt`
- drwxr-xr-x. 3 root root 18 Jan 27 16:15 `gurudeen`
- -rw-r-----. 1 root root 22 Feb  4 21:50 `notes.txt`
- drwxr-xr-x. 2 root root  6 Feb  4 22:25` project`
- -rwxr-xr-x. 1 root root 21 Feb  4 21:51 `script.sh`
- drwxr-xr-x. 3 root root 28 Jan 29 21:59 `trainwithsubham`

```bash
## Commands Used
touch devops.txt
  850  echo " this notes,txt file " > notes.txt
  851  vim script.sh
  852  ls -l
  853  cat notes.txt 
  854  vim -r script.sh 
  855  vim  script.sh 
  856  head -5 /etc/passwd
  857  tail -5 /etc/passwd
  858  ls -l devops.txt 
  859  ls -l notes.txt 
  860  ls -l notes.txt  script.sh 
  861  chmod +x script.sh 
  862  chmod -wx devops.txt 
  863  ls -l devops.txt 
  864  ls -l 
  865  cat >> devops.txt 
  866  exit 
  867  ./script.sh 
  868  chmod -x script.sh 
  869  ./script.sh 
  870  chmod +x script.sh 
  871  chmod 640 notes.txt 
  872  mkdir 755  project
  873  ls -l project/
  874  ls -ld project/
  875  ls
  876  chmod 755 project/
  877  rmdir 755/
  878  ls
  879  ls -ld
  880  ls -l
  881  history 
```
---
## What I Learned
- today i have leraned creating empty files and with content file by using `touch`, `cat` and `echo` command.
- modification of file like append with` >,>>,2> and 2>>`
- view the content in files and using these command `head,tail,vim and cat etc.`
- understand the `default permission, special permissions and acls(access control list )`

- 
