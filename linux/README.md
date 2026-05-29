# Linux 📖

> Hey there — this is my personal study database and search hub.  
> It covers Linux from **basic to advanced**, and it's built to help me (and maybe you) find answers fast.

---

## 🎯 When to Use This

- You're stuck on a challenge and can't figure it out
- You forgot how a command works
- You want to see examples (including mistakes I've made 😅)

**Example:** Having trouble using `grep` to search inside a file?  
Come here → check the options → study them → go back and try again.

---

## 🧭 Recommended Learning Path

This file is your **quick reference + database**, not your main teacher.

For deeper understanding, I highly recommend:

- 🎓 **pwn.college** — amazing for command fundamentals
- 🧪 **Hack The Box** — real-world practice
- 🐣 **TryHackMe** — beginner-friendly Linux rooms
- 🎮 **OverTheWire (Bandit)** — learn by solving levels

---

## 🔍 Quick Search Guide (When You're Stuck)

| Problem | What to Do |
|---------|------------|
| Stuck on `grep` | Check pwn.college or Bandit levels |
| Forgot a flag | Use `man <command>` or `--help` first |
| Made a mistake | Check my notes — I've probably done it too 😅 |

---

## 📖 How to Use This File

### 1. 🔍 Search by *What You Want to Do*, Not by Command Name

- Want to create a file? Search **`create file`**, not `touch`
- This finds commands by *purpose*, not just name

### 2. 🔤 Everything Is in Alphabetical Order

- Commands and concepts from **A to Z** for easy scanning
- **Pro tip:** `Ctrl + F` or `/` to jump instantly

### 3. 🧠 Don't Panic — Advanced Stuff May Appear Early

Because of A–Z order, complex topics (like `📁 Absolute Path`) might show up before basic commands.

**But you're fine because:**
- Each concept is **self-contained**
- Cross-references point to related sections
- Just keep reading — it'll click later

### 4. 🧩 How Each Topic Is Structured

Every entry follows the same pattern:

| Section | What You'll Find |
|---------|-----------------|
| **Heading** | What you want to do (e.g., "Create a file") |
| **Command** | The actual terminal command |
| Description | If needed |
| Basic usage | Simple examples |
| Advanced options | Sometimes |
| Common mistakes | What I've messed up 😅 |

> 💡 **Remember:** Search the **heading**, not the command.  
> Example: want to create a file? Search **"create file"** → you'll find `touch`

### 5. 🎨 Icon Guide

| Icon | Meaning |
|------|---------|
| 📁 | Key concept (important to understand) |
| *(no icon)* | Just a regular command |

### 6. 🚀 Ultimate Tip

Go inside the command file and press **`Ctrl + K`** to search for whatever you need.

---

## 🐧 Linux Terminal Basics (Before You Start)

### How Linux Commands Work

You'll be using a terminal (Kali, Ubuntu, any distro — same rules).

**First thing to know:** Linux is **case sensitive**
- `cp` and `CP` are different
- `Hello` vs `hello`? Also different.

### Understanding the Terminal Prompt

Let's look at an example:

```bash
bro@kali:~$ mkdir -p time/no_time
```

Here's what each part means:

| Part | Meaning |
|------|---------|
| `bro` | Your username |
| `kali` | Hostname (machine name) |
| `~` | Current directory (`~` = home folder) |
| `$` | Regular user (not root) |
| `#` | Root user (if you see this instead) |

**Quick reference:**
- `~` = home directory (`/home/bro`)
- `/` = root directory (top of the filesystem)
- `$` = normal user
- `#` = root user (be careful!)

**In the example above:**
- `mkdir` = command to create a directory
- `-p` = option (creates parent folders if they don't exist)
- `time/no_time` = directories to create

> ⚠️ **Note:** If you see `root@kali:/#` instead, you're the root user working from the `/` directory.

### Command Structure

A basic command looks like this:

**Example 1 — single argument:**

```bash
echo "Hello"
```

- `echo` = command (what you want to do)
- `Hello` = argument (what you're doing it to)

**Example 2 — multiple arguments:**

```bash
echo Hello man
```

- `Hello` and `man` = two arguments
- You can give multiple arguments to most commands

---

## 🎩 Handy Trick: Command History

We're going to type a lot of commands. Don't retype everything!

Just press `↑` (arrow up) or `↓` (arrow down) to scroll through your previous commands.

---

> Study → Get Stuck → Check Source → Come Here → Go Back → Break More Things 💥
>
> Happy hacking! 🐧
