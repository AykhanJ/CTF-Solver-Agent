# 🤖 CTF Agent — Automated Capture The Flag Solver

An LLM-powered agent that autonomously solves CTF (Capture The Flag) challenges using GPT-4o. The agent reasons through each challenge step by step, executes shell commands, and extracts flags — without ever directly reading protected files. This agent can be improved for other specific vulnerabilities and can be updated for better accuracy. 

---

## 📋 Overview

This agent is designed to tackle two categories of CTF challenges:

| Challenge | Type | Technique |
|---|---|---|
| `follow_that_pwnie` | Python script exploitation | Arbitrary code execution via `eval` injection |
| `sharpgame` | Binary exploitation | Buffer overflow / return-address hijacking |

The agent is given a target directory, reasons about the challenge structure, issues shell commands, and iterates until it finds the flag or hits the step limit.

---

## 🗂️ Project Structure

```
agent.py          # Main agent script
challenges/
├── follow_that_pwnie/    # Python eval-injection challenge
│   ├── challenge.json
│   └── follow_that_pwnie.py
└── sharpgame/            # Binary exploitation challenge
    ├── challenge.json
    └── sharpgame         # ELF binary
```

---

## ⚙️ How It Works

1. **Agent is initialized** with a target challenge folder.
2. A **system prompt** instructs GPT-4o to act as a penetration tester, explains both challenge types, and enforces response formatting as JSON.
3. In a loop (up to `MAX_STEPS = 30`), the agent:
   - Sends the conversation history to GPT-4o.
   - Receives a JSON action (`run_command` or `finish`).
   - Executes the command via a sandboxed shell runner.
   - Appends the output back to the conversation.
4. If a **flag pattern** (`csawctf{...}`, `flag{...}`, `FLAG{...}`) is detected in any command output, the agent exits immediately and reports the flag.

### Response Format (Agent Output)

```json
{
  "action": "run_command",
  "command": "ls -la follow_that_pwnie/",
  "reasoning": "Exploring the directory structure to understand the challenge",
  "flag": null
}
```

```json
{
  "action": "finish",
  "reasoning": "Found the flag via buffer overflow exploit",
  "flag": "FLAG{example}"
}
```

---

## 🔐 Security Constraints

The agent enforces the following rules to keep challenges fair:

- **Direct flag reads are blocked.** Commands containing `flag` combined with read utilities (`cat`, `grep`, `strings`, `vim`, etc.) are rejected.
- **Flags must be extracted through the program**, not from the filesystem directly.
- **Loop prevention:** If the agent repeats the same command 3 times in a row, it is forced to try something new.

---

## 🧩 Challenge Walkthroughs

### Challenge 1 — `follow_that_pwnie` (Python Exploitation)

The Python script uses `eval()` on unsanitized user input, creating an arbitrary code execution vulnerability.

**Agent strategy:**
1. Read `challenge.json` and `follow_that_pwnie.py` to understand the structure.
2. Use `ls` to map out directories and locate flag files.
3. Craft a payload that abuses `eval` to print the flag contents at runtime (e.g., via `__import__('os').popen(...).read()`).
4. Pass the extracted flag value back as the program's password input to reach the `"You win! 🐴"` output.

### Challenge 2 — `sharpgame` (Binary Exploitation)

The binary contains a classic buffer overflow vulnerability.

**Agent strategy:**
1. Run `file` and `objdump` to understand the binary's architecture and control flow.
2. Use `pwntools` (`ELF().symbols`) to locate win functions (e.g., `cat_flag`).
3. Generate a cyclic pattern to calculate the overflow offset.
4. Craft a payload that overwrites the return address to redirect execution to the win function.
5. Extract the flag printed by the win function.

**Available tools for binary challenges:**
- `pwntools`
- `radare2` / `r2` / `r2pm`
- `objdump`
- `meson` / `ninja`

---

## 🚀 Setup & Usage

### Prerequisites

```bash
pip install openai pwntools
```

Ensure the following tools are installed on your system:
- `radare2`
- `objdump` (part of `binutils`)

### Environment Variables

```bash
export OPENAI_API_KEY=your_api_key_here
```

### Running the Agent

```bash
python agent.py <challenge_folder>
```

**Examples:**

```bash
# Solve the Python injection challenge
python agent.py challenges/follow_that_pwnie

# Solve the binary exploitation challenge
python agent.py challenges/sharpgame
```

---

## 🛠️ Configuration

| Constant | Default | Description |
|---|---|---|
| `MAX_STEPS` | `30` | Maximum reasoning iterations before giving up |
| `MODEL` | `gpt-4o` | OpenAI model used for reasoning |

These can be changed at the top of `agent.py`.

---

## ⚠️ Disclaimer

This tool is intended for **educational purposes and authorized CTF competitions only**. Do not use it against systems you do not have explicit permission to test.
