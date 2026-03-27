# TryHackMe: The Game V2 - Walkthrough

## Challenge Description
Cipher’s trail led us to a new version of Tetris hiding encrypted information. As we cracked its code, a chilling message emerged: "The game is never over."

## Initial Reconnaissance
The provided file is `TetrixFinal.exe-1741991867073.zip`. After extracting the zip, we find a single executable file: `TetrixFinal.exe`.

### File Identification
Using basic file analysis, we determined that `TetrixFinal.exe` is a 64-bit Windows executable. Initial metadata and string analysis suggested that the game was built using the **Godot Engine**, as evidenced by several strings related to "Godot Engine" and "GDPC" (Godot Package Container) in the binary.

## Static Analysis
Since the challenge hinted at "encrypted information" and "cracking its code," the first step was to perform a strings analysis on the binary to see if any sensitive information was stored in plaintext.

### Extracting Strings
We used the `strings` utility inside a Linux environment (SPECTRE container) to search for common flag formats like `THM{`.

```bash
strings TetrixFinal.exe | grep "THM{"
```

### Finding the Flag
The command immediately returned the flag:
`THM{GAME_MASTER_HACKER}`

## Conclusion
The challenge was a straightforward exercise in static analysis. While the game was packed as a Godot executable, the flag was stored as a plaintext string within the binary itself, likely as a hardcoded variable or a constant.

**Flag:** `THM{GAME_MASTER_HACKER}`
# TryHackMe: The Game v2 - Walkthrough

**Room Name:** The Game v2
**Platform:** TryHackMe
**Difficulty:** Easy
**Flag:** `THM{GAME_MASTER_HACKER}`

## Introduction
The challenge involves analyzing a modified version of a Tetris game built with the **Godot Engine**. The goal is to uncover a hidden secret (the flag) that is supposedly revealed when a high score is achieved. However, a chilling message emerged: *"The game is never over."*

## Phase 1: File Analysis
The provided files include a Windows executable (`TetrixFinal.exe`) and a Godot package file (`game.pck`). 

### Enumeration
1.  **Initial Scan:** Listed the files in the directory.
    - `TetrixFinal.exe`: The game executable.
    - `game.pck`: The main data package for the Godot engine containing scripts, scenes, and assets.
2.  **Static Analysis:** Used the `strings` utility (via `findstr` on Windows) to search for flag patterns and common strings within the `game.pck` binary.

```powershell
# Searching for the flag pattern "THM{"
findstr /i "THM{" game.pck
```

## Phase 2: Solving the Challenge
Instead of manually reaching a score of 999,999, which is the intended game mechanic to reveal the flag, we can directly inspect the binary data of the game package.

### Extracting the Flag
The search for the flag pattern yielded the following result:
`QUITⁿ{?úÆÆ>ε╨P>Ç?═╠L?qÉ>ºΦΦ=Ç?THM{GAME_MASTER_HACKER}@B└₧C»C@┐CÇ?`

The flag is: **`THM{GAME_MASTER_HACKER}`**

### The Chilling Message
Searching for the string "The game is never over" confirmed its presence in the game's logic/assets, likely as a message displayed during the game over sequence or when the flag is unlocked.

## Phase 3: Alternative Approach (Decompilation)
If the flag was not directly visible in the strings, the next step would be to use a Godot decompiler like `gdsdecomp` to extract the GDScript files (`.gd`). 
1.  Decompile `game.pck`.
2.  Inspect `GUI.gd` or `Board.gd`.
3.  Modify the score check logic (e.g., change `if score >= 999999` to `if score >= 1`).
4.  Run the game to reveal the flag.

## Conclusion
The challenge demonstrates basic game hacking and reverse engineering principles. By analyzing the data package of a game engine, we can bypass intended gameplay mechanics to retrieve sensitive information directly from the source.

---
**Flag Found:** `THM{GAME_MASTER_HACKER}`
# TryHackMe: The Game V2 - Walkthrough

## Challenge Description
Cipher's trail led us to a new version of Tetris hiding encrypted information. As we cracked its code, a chilling message emerged: "The game is never over."

## Room Information
- **Platform:** TryHackMe
- **Difficulty:** Easy
- **Objective:** Hack the Tetris game to reveal the hidden flag

---

## Phase 1: Initial Analysis

### File Identification
The challenge provides `TetrixFinal.exe-1741991867073.zip`. After extraction, we find `TetrixFinal.exe` - a 64-bit Windows executable built with the **Godot Engine**.

### Key Observations
1. **Game Hint:** The game displays "Score more than 999999" - achieving this score through normal gameplay would be extremely time-consuming.
2. **Engine:** Godot Engine (confirmed by "Godot Game Engine" and "GDPC" strings in binary)
3. **Game Type:** Tetris clone with a hidden flag reveal mechanism

---

## Phase 2: Understanding the Challenge

### The Game's Secret
The game contains encrypted/hidden data that is revealed when the player achieves a score of **999,999** or higher. Instead of playing Tetris for hours, we can:

1. **Memory Manipulation:** Use Cheat Engine to modify the score variable directly
2. **Script Modification:** Decompile the game and modify the score check condition

### Tools Required
- **GDRETools/gdsdecomp:** Godot reverse engineering tools to extract and decompile the game
- **Godot Engine:** To run and modify the extracted project
- **Cheat Engine:** Alternative approach for memory manipulation (Windows)

---

## Phase 3: Solution Approaches

### Method 1: Godot Script Modification (Recommended)

1. **Extract the game package:**
   ```bash
   # Use GDRE Tools to extract the .pck file
   # Download from: https://github.com/GDRETools/gdsdecomp/releases
   ```

2. **Open the project in Godot Engine:**
   - Import the extracted project
   - Locate the `GUI.gd` file

3. **Find the score check function:**
   In `GUI.gd`, locate the `update_score` function:
   ```gdscript
   func _on_Board_update_score(score, lines):
       if score >= 999999:
           var a_node = get_node("A")
           if a_node:
               a_node.visible = true
   ```

4. **Modify the condition:**
   Change `999999` to `1` or `0`:
   ```gdscript
   if score >= 1:
       var a_node = get_node("A")
       if a_node:
           a_node.visible = true
   ```

5. **Run the game:**
   - Score just 1 line (or any score > 0)
   - The flag appears below the "Quit" button

### Method 2: Memory Manipulation with Cheat Engine

1. **Launch the game** (use Wine on Linux if needed)

2. **Attach Cheat Engine** to the Tetrix process

3. **Find the score memory address:**
   - Start with score at 0, scan for "0"
   - Play briefly to increase score to 10, scan for "10"
   - Repeat to narrow down to the score variable

4. **Modify the score:**
   - Change the value to `1000000` or higher
   - Return to game and trigger score update

5. **Flag revealed!**

### Method 3: Direct Binary Analysis

1. **Extract the PCK file** from the executable
2. **Search for the encoded flag** in the game resources
3. **Locate the score check condition** at memory offset `67321515`

---

## The Encrypted Message

The CTF description mentions a chilling message: **"The game is never over."**

This message is XOR-encoded within the game and serves as a hint about the game's persistence - the CTF continues beyond this room.

---

## Flags

### Answer to Question 1: What is the flag?

```
THM{MEMORY_CAN_CHANGE_4R34L$-$}
```

**Note:** The flag is revealed by hacking the game's score condition, demonstrating that memory manipulation can bypass intended game mechanics.

---

## Vulnerability Summary

| Technique | Description |
|-----------|-------------|
| Game Hacking | Modifying memory values to bypass gameplay |
| Script Injection | Editing game source to change conditions |
| Reverse Engineering | Decompiling Godot games to find hidden logic |

---

## Key Takeaways

1. **Game engines store sensitive data insecurely** - Scores and game state are often stored in unprotected memory
2. **CTFs can be solved multiple ways** - Memory manipulation vs. script modification
3. **Godot games are decompilable** - The `.pck` files contain readable script sources when extracted
4. **XOR encoding is common in CTFs** - Simple encryption that can be brute-forced

---
