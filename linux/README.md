# Linux 📖

Hello — as this repo is my study database and search hub, this file covers Linux in general. It includes both basic and advanced commands.

---

## How to Use This File

1. The commands and keywords you'll learn are mostly in **alphabetical order** for easier use and search. Or you can just      jump in if you already know what you're looking for. As its in alphabetic order, some more advance concept may come         first or you may not understand something, just use the file i will mention it in their own section.
   
2. Each entry includes:
   - Description (If needed)
   - Basic command use 
   - Common mistakes I've made
   - Sometimes advanced arguments/options

3. Concepts (not just commands) are included — like how a command works under the hood.

---

## Linux Terminal Basics (Before You Start)

### How Linux Commands Work

You'll be using a terminal (Kali Linux or any other distribution — same rules apply).

**First thing:** Linux is **case sensitive**  
- `cp` and `CP` are different. So is `Hello` vs `hello`.

### Understanding the Terminal Prompt

Before running commands, know what you're looking at:

bro@kali:~$ echo "Hello"


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
- `echo` = command
- `"Hello"` = argument

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
