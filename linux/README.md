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
|---------|-------------|
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
|---------|------------------|
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
