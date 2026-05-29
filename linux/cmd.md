# Linux Commands Reference

---

## 📁 Absolute Path vs Relative Path

**Description:**  
First thing you need to know about navigating Linux. **Always be careful about which path to use for all commands.**

- **Absolute path** → Starts from the root `/`  
  Example: `/home/user/Documents/file.txt`

- **Relative path** → Starts from where you currently are `./`  
  Example: `./file.txt` or just `file.txt` (if it's in the current directory)

**Important Security Rule:**

Normally you can `cat filename` and it works fine.  
But when you **execute a shell file**, Linux requires an absolute or explicit path for security reasons.

Correct way to run a script in the current directory:

```bash
./filename
```

**Trick & problem I faced:**

When you need to access a file using an absolute path from your home directory, use the following pattern:

```bash
command ~/name
```

- `command` = what you want to do
- `~` = your home directory
- `/` = absolute path separator
- `name` = your file name

---

## Copy File

**cmd:**

```bash
cp source destination
```

---

## Create File

**cmd:**

```bash
touch filename
```

---

## Difference Between Files

**cmd:**

```bash
diff file1 file2
```

**Output example:**

```bash
2c2
< world
---
> universe
```

| Notation | Meaning |
|----------|---------|
| `1a2` | After line 1 of file1, add line 2 of file2 |
| `2c2` | Change line 2 in file1 to match line 2 in file2 |
| `3d2` | Delete line 3 from file1 to match file2 starting at line 2 |

**Key:**
- `a` = add
- `c` = change
- `d` = delete
- `<` = first file
- `>` = second file

**Trick & problem I faced:** Trying and understanding the different output formats — what changed and how to read it properly.

---

## 📁 Directory

| Symbol | Meaning |
|--------|---------|
| `.` | Current directory |
| `..` | Parent directory |
| `~` | Home directory shortcut |
| `-` | Previous directory (used with `cd -`) |

---

## Find File

**cmd:**

```bash
find [starting_path] [options] [expression]
```

**Options:**

| Option | Description |
|--------|-------------|
| `-name` | Search by exact filename |
| `-iname` | Search by filename (case-insensitive) |
| `-type` | Search by type (`f` = file, `d` = directory) |

---

## 📁 Globbing

**Description:**

*(coming soon)*

---

## Help

**cmd:**

```bash
command --help
```

Other help options:

| Option | Notes |
|--------|-------|
| `-h` | Human-readable — only works for some commands like `du`, `df` |
| `-?` | Some commands only, like `rm` |
| `help` | Shell built-ins only |

---

## List All Files

**cmd:**

```bash
ls -option
```

**Options:**

| Option | Description |
|--------|-------------|
| `-a` | List all files, including hidden ones |

---

## Make Directories

**cmd:**

```bash
mkdir -option dirname
```

**Options:**

| Option | Description | Example |
|--------|-------------|---------|
| `-p` | Create parent directories as needed | `mkdir -p parent/child/grandchild` |
| `-m` | Set permissions (see permissions section) | `mkdir -m 755 myfolder` |
| `-v` | Verbose — show each created directory | `mkdir -v folder1 folder2` |

---

## Make File

**cmd:**

```bash
touch filename
```

---

## Manual for Commands

**cmd:**

```bash
man [section] command_name
```

For most cases, just use:

```bash
man command_name
```

**How to navigate man pages:**

| Key | Action |
|-----|--------|
| `↑` / `↓` | Scroll line by line |
| `PgUp` / `PgDn` | Scroll page by page |
| `/keyword` | Search forward |
| `?keyword` | Search backward |
| `n` | Next search result |
| `N` | Previous search result |

You can even run `man` on itself:

```bash
man man
```

---

## Move File

**cmd:**

```bash
mv source destination
```

---

## Read File

**cmd:**

```bash
cat filename
```

---

## Remove File

**cmd:**

```bash
rm filename
```

---

## Search Content Inside File

**cmd:**

```bash
grep search_string /path/to/file
```

**Options:**

| Option | Description |
|--------|-------------|
| `-i` | Case-insensitive search |
| `-r` | Recursive search through directories |
| `-n` | Show line numbers |
| `-v` | Show lines that **don't** match |
| `-l` | Show only filenames with a match |

**Trick & problem I faced:**

```bash
grep "search string" /path/to/file
```

Use `""` if your search string contains spaces.

---

## 📁 Symbolic Link & Hard Link

> Mostly focused on symbolic links for now.

**Description:**  
A shortcut or pointer to another file or directory — like a desktop shortcut in Windows or an alias in macOS.

**cmd:**

```bash
ln -s [target_file_or_directory] [link_name]
```

**Example — create a symlink to a file:**

```bash
ln -s /home/user/Documents/notes.txt mynote
```

**How to identify symlinks:**

List with detail (symlinks usually appear in cyan/blue):

```bash
ls -l mynote
```

Output:

```
lrwxrwxrwx 1 user user 24 Dec 15 mynote -> /home/user/Documents/notes.txt
^
'l' means link
```

Find all symlinks in current directory:

```bash
find . -type l
```

**Tricks:**

- Use **absolute paths** for reliable symlinks — relative paths can break if you're in the wrong directory.
- Don't forget `-s` — without it, `ln` creates a **hard link**, which is not what you usually want.
