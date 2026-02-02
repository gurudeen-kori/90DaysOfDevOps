
# Day_06 Of 90DaysOfDevOPs Challenge 
## Create an empty file named notes.txt
```bash
touch notes.txt
```

# Write text to the file (overwrites if file exists)
```bash
echo "Hello, this is the first line." > notes.txt
echo "This is the second line." >> notes.txt
```
# Append new lines to the file
``` bash
echo "This line is line 2." >> notes.txt
echo "this is  line 3." >> notes.txt

```
# view content in notes.txt 
```bash
head -2 notes.txt
tail -2 notes.txt
```

# tee command 
```bash
tee -a notes.txt
```
```bash
# Display the content of the file
cat notes.txt
```
```bash
less notes.txt   # Scroll through the file
more notes.txt   # Scroll page by page

```


