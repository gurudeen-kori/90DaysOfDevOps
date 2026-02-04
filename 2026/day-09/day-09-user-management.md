# Day 09 Challenge

## Users & Groups Created
- Users: tokyo, berlin, professor, nairobi
- Groups: developers, admins, project-team

## Group Assignments
```bash
groups tokyo
groups berlin
```

## Directories Created
```bash
ls -l /opt/team-workspace
```
## Commands Used
```bash
  adduser tokyo
  794  adduser berlin
  795  adduser professor
  796  groupadd developer admin
  797  groupadd developer 
  798  groupadd admin
  799  tail -3 passwd
  800  tail -3 /etc/passwd
  801  tail -3 /etc/group
  802  cd
  803  usermod -aG developer tokyo 
  804  usermod -aG developer berlin 
  805  usermod -aG  developer admin berlin 
  806  usermod -aG  developer,admin berlin 
  807  usermod -aG  admin professor 
  808  mkdir -p /opt/dev-project
  809  ll
  810  ls /opt/dev-project/
  811  ls -a /opt/dev-project/
  812  ll
  813  cd /opt/
  814  ll
  815  chown root:developer dev-project/
  816  ls -l
  817  chgrp developer dev-project/
  818  ls -l
  819  chmod 775 dev-project/
  820  ls -l
  821  su - tokyo
  822  pwd
  823  cd /dev/
  824  pwd
  825  ls /
  826  tree /
  827  tree /opt/
  828  su - tokyo
  829  cd /opt/dev-project/
  830  ls
  831  touch root.txt
  832  exit
  833  su - tokyo
  834  su - berlin 
  835  cd /opt/dev-project/
  836  ll
  837  cat > test.txt 
  838  ll
  839  chmod 700 /opt/dev-project/
  840  su - berlin 
  841  chmod 744 /opt/dev-project/
  842  su - berlin 
  843  chmod 764 /opt/dev-project/
  844  su - berlin 
  845  chmod 775 /opt/dev-project/
  846  su - berlin 
  847  adduser nairobi -m 
  848  groupadd project-team
  849  usermod -aG project-team nairobi,tokyo
  850  usermod -aG project-team nairobi
  851  usermod -aG project-team tokyo
  852  cd
  853  mkdir -p /opt/team-workspace
  854  ll /opt/team-workspace/
  855  cd /opt/
  856  ll
  857  chgrp project-team team-workspace/
  858  ll
  859  chmod 775 team-workspace/
  860  ll
  861  su - nairobi 
  862  history 

```
## What I Learned
---
- I have learned that users are individual accounts in Linux that allow people to log in and access system resources.
- Each user has a unique user ID (UID) and can belong to one or more groups.

- Groups help manage permissions collectively, making it easier to control access to files and directories.

- I learned how to create a new user using the useradd command.

- I learned how to modify user properties with the usermod command, such as changing group membership or home directory.

- I learned how to delete users safely with the userdel command.

- I understood the importance of assigning users to appropriate groups to enforce the principle of least privilege.

- I learned how to create new groups using the groupadd command.

- I learned how to change a user’s group membership using usermod or gpasswd.

- I learned how to set and update user passwords using the passwd command.

- I understood how group ownership affects file and directory permissions.

- I learned that proper user and group management is crucial for system security and organization.

- I also understood that regular monitoring of users and groups helps prevent unauthorized access.

- Overall, user and group management is a fundamental skill for Linux system administration.
---