# for_loop.sh
```bash 
#!/bin/bash 

fruits=("banana" "apple" "greps" "mango" "papapya")

for fruits in "${fruits[@]}"
do
        echo " fruit : $fruits"

done
---
```
---
- in this script i have learned about for loop and 
# count.sh 
```bash 
!/bin/bash 

for num in {1..10}
do 
	echo " number : $num "
	 
done

read -p " enter username " username
read -p "passwd" pass
sudo useradd $username -p $pass
```
# countdown.sh
```bash 
#!/bin/bash

# Take number input from user
read -p "Enter a number: " num

# Countdown using while loop
while [ "$num" -ge 0 ]
do
    echo "$num"
    ((num--))
done

echo "Done!"
```
# greet.sh
```bash 
#!/bin/bash

read -p " enter your name " s

if [ -z "$s" ]; then
        echo " Usage: ./greet.sh "
        exit 1
else
        echo " hello ! $s "

fi
```

Install Packages via Script
```bash 
!/bin/bash

packages=("nginx" "curl" "wget")

for package in "${packages[@]}"; do
    echo "Checking $package..."

    if command -v dpkg >/dev/null 2>&1; then
        if dpkg -s "$package" >/dev/null 2>&1; then
            echo "$package is already installed. Skipping."
        else
            echo "$package is not installed. Installing..."
            sudo apt-get install -y "$package"
        fi

    elif command -v rpm >/dev/null 2>&1; then
        if rpm -q "$package" >/dev/null 2>&1; then
            echo "$package is already installed. Skipping."
        else
            echo "$package is not installed. Installing..."
            sudo yum install -y "$package"
        fi

    else
        echo "No supported package manager found."
        exit 1
    fi

    echo
done
```

~                                                                                                                                                                                                            
~                      
