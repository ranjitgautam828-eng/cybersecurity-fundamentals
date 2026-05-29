## 📁 Absolute Path vs Relative Path

**Description:**  
First thing you need to know about navigating Linux.

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

##
