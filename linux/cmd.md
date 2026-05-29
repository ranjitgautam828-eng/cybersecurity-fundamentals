## 📁 Absolute Path vs Relative Path

**Description:**  
First thing you need to know about navigating Linux. **Always be careful for location use for all cmd**

- **Absolute path** → Starts from the root `/`  
  Example: `/home/user/Documents/file.txt`

- **Relative path** → Starts from where you currently are `./`  
  Example: `./file.txt` or just `file.txt` (if it's in the current directory)

**Important Security Rule:**

- Normally you can `cat filename` and it works fine.  
But when you **execute a shell file**, Linux requires an absolute or explicit path for security reasons.

- **Correct way to run a script in current directory:**  

    ```bash
    ./filename
     ```

**Trick & problem I faced**

When you have to access the file from your home directory but absolute path not relative,you the following example.

**Example of absolute path from home directory:**

```bash
  command ~/name
 ```
- `command` = what you want to do
-  `~` = your home directory
- `/` = absolute path
- `name` = your file name

---

## Copy file

**cmd**: 
 ```bash
  cp source destination
 ```
---

## Create file

**cmd**: 
 ```bash
    touch filename
 ```

---

## Difference between file 

**cmd**:  
 ```bash
    diff file1 file2
 ```
**output example**:
```bash
2c2
< world
---
> universe
```

1a2-> means after line 1 of file1, add line 2 of file2 

2c2-> means “change line 2 in first file to match line 2 in second file”

3d2 -> delete line 3 from first file to match second file starting at line 2.

**Trick & problem I faced**: Trying and understandig different output, how it works, or what changed properly.

a = add

c = change

d = delete
< = first file, > = second file

---

## 📁Directory

-Current directory -> .

-Parent directory -> ..

-Home directory shortcut -> ~

-Previous directory -> - (used with cd -)

---

## Help 
**cmd**: 
 ```bash
    command --help
 ```

You can use other cmd as: 

-h (human-readable) -> risk as it only applicable for some cmd like du (disk usage), df (disk free space)

-? -> Some commands only like rm 

help (Shell built-ins only)

---

## Find file
**cmd**: 
 ```bash
    find [starting_path] [options] [expression]
 ```
**Option**:

-name: Search by exact filename

-iname: Search by filename (case-insensitive)

-type: Search by type (f=file, d=directory)

---

## 📁 Globbing

**Description**:


---

## list all the file

**cmd**: 
 ```bash
    ls -option
 ```
**Option**:

-a: to list all file even hidden one

---

## Make directories

**cmd**: 

 ```bash
    mkdir - option filename
 ```
**Option**:
-p -> Create parent directories as needed
```bash
    mkdir -p parent/child/grandchild
 ```
-m -> Set permissions -> more in persmission section
```bash
    mkdir -m 755 myfolder 
 ```
-v -> Verbose – show each created directory
```bash
    mkdir -v folder1 folder2
 ```

---
## Make file

**cmd**: 

 ```bash
    touch filename
 ```

--- 

## Manual for cmd

**cmd**: 

 ```bash
    man [section] command_name
 ```
 **To use**:
 
 Only use section if you want otherwise just below cmd is fine:
  ```bash
    man command_name
 ```
 You can scroll man pages with the arrow keys (and PgUp/PgDn) and search with /. After searching, you can hit n to go to 
 the next result and N to go to the previous result. Instead of /, you can use ? to search backwards!

 You can use man on man 
  ```bash
    man man
 ```
 You can use website for more detail if your are intrested in this.
 
--- 
## Move file

**cmd**: 

 ```bash
    mv source destination
 ```

---

## Read file 

**cmd**: 
 ```bash
    cat filename
 ```

---

## Search content inside file

**cmd**: 

 ```bash
   grep Search_string /path/file
  ```

**Option**:

-i → case insensitive search

-r → recursive search through directories

-n → show line numbers

-v → show lines that DON'T match

-l → show only file names with match

**Trick & problem**:
```bash
 grep "search string" /path/to/file
 ```
-> use "" if your string has space

---

## 📁 Symbolic Link & Hard link

-> mostly focused on symbolic link for now

**Description**:
A shortcut or pointer to another file or directory. Like a desktop shortcut in Windows or alias in macOS.

**cmd**: 
 ```bash
   ln -s [target_file_or_directory] [link_name]
 ```
example:
Create a symlink to a file
 ```bash
   ln -s /home/user/Documents/notes.txt mynote
 ```

**How to identify symlinks**:
- List with color (symlinks usually cyan/blue)
 ```bash
   ls -l mynote
 ```

- Output:

lrwxrwxrwx 1 user user 24 Dec 15 mynote -> /home/user/Documents/notes.txt

^ 'l' means link

- Find all symlinks
 ```bash
   find . -type l
 ```

**Trick**:

-Use absolute paths for reliable symlinks as i sometime have problem when using relative link as we may usewrong relative path.
-s -> if you forgot this option to use then it create a hard link which we dont want.

---

## Remove file

**cmd**: rm
 ```bash
    rm filename
 ```

