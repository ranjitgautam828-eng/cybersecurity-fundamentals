# Linux 📖

Hello — as this repo is my study database and search hub, this file covers Linux in general. It includes both basic and advanced commands.

---

## How to Use This File

1.📝 **Search by Description, Not Command**. 
  - If you want command to create a file? Search 'create file', not 'touch'

2. 🔤 **Alphabetical Order for Easy Scanning**.
    - Commands and concepts are arranged **A to Z** for easy use.
    - **Pro tip:** Use your editor's search (`Ctrl+F` or `/`) to jump instantly.
  
4. 🧠 **Advanced Concepts May Appear Early**
      - Some complex ideas come first alphabetically (like `📁 Absolute Path` before basic commands).
 
  **Solution:** 
    
    - Each concept is **self-contained**
    
    - Cross-references point to related sections
    
    -Just keep reading — it will make sense later
   
 5. 🧩 Concepts Included, Not Just Commands
   - Description (If needed)
   - Basic command use
   - Sometimes advanced arguments/options
   - Common mistakes I've made
   
  6. Icon Meaning
      - 📁 means keyconcept in the file. otherwise its just command.

---

## Linux Terminal Basics (Before You Start)

### How Linux Commands Work

You'll be using a terminal (Kali Linux or any other distribution — same rules apply).

**First thing:** Linux is **case sensitive**  
- `cp` and `CP` are different. So is `Hello` vs `hello`.

### Understanding the Terminal Prompt

Before running commands, know what you're looking at:

bro@kali:~$ mkdir -p time/no_time


| Part | Meaning |
|------|---------|
| `bro` | Username (you) |
| `kali` | Hostname (machine name) |
| `~` | Current working directory (`~` means home folder) |
| `$` | Regular user (not root) |
| `#` | Root user (if you see this instead of `$`) |

**Quick reference:**
- `~` = home directory (`/home/bro`)
- `/` = root directory (if you're root user)
- `$` = normal user
- `#` = root user

**In the example:**
- `mkdir` = command to create directory
-p = option (tells mkdir to make parent folders too) if time directory is not there
- time/no_time = directories to create
*Note: If you see `root@kali:/#` instead, you're the root user working from `/` directory.*

### Command Structure

A basic command looks like this:

Example:

1. echo "Hello"

→Here, `echo` = the **command** (what you want to do) ,`Hello` = the **argument** (what you're doing it to)

2. echo Hello man

→ Here, `Hello` and `man` are two arguments. You can give multiple arguments:

### Handy Trick: Command History

We're going to use a lot of commands. Don't retype everything —  
Just press **arrow up** or **arrow down** to scroll through what you typed before.


---

## When to Use This

- You're stuck on a challenge and can't complete it  
- Example: having trouble using `grep` to search inside a file? Come here, check the options, study them, then go back and try again.

---

## Recommended Learning Path

I highly recommend going through **pwn.college** for deeper understanding.  
Use this file as your **database + quick reference**, not your primary teacher.

---

## 🔍 Source (Where I Learned This)

Use these to search for more details or practice:

| Source | Best For | Search Tip |
|--------|----------|-------------|
| pwn.college | Deep understanding of commands | Search their dojo for specific flags |
| Hack The Box | Real-world practice | Use `man` pages first, then compare |
| TryHackMe | Beginner-friendly rooms | Check their Linux rooms |
| OverTheWire (Bandit) | Learning by solving | Each level teaches a new command |

**Quick Search Guide:**  
- Stuck on `grep`? → Check pwn.college or Bandit levels  
- Forgot a flag? → Use `man <command>` or `--help` first  
- Made a mistake? → It's probably in my notes too 😅

---

> Study → Get Stuck → Check Source → Come Here → Go Back → Break More Things 💥
