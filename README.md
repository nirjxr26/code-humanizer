# code-humanizer

A prompt skill that explains any codebase in plain, developer-friendly language — fast, clear, and without the usual AI slop.

Paste code. Pick a mode. Get an explanation that actually tells you something.

---

## Installation

### Claude Code

Clone directly into Claude Code's skills directory:

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/nirjxr26/code-humanizer.git ~/.claude/skills/code-humanizer
```

Restart Claude Code after installation.

---

## Usage

### Claude Code

```txt
/code-humanizer
```

Or:

```txt
/code-humanizer

/smell
[paste your code here]
```

## What it does

Most AI code explanations redescribe syntax you can already read. This one doesn't.

`code-humanizer` explains **intent**, **decisions**, and **failure paths** — the things the code itself doesn't say. It works for two audiences: developers who need to understand code fast, and students who need to understand what a project is actually doing and why.

---

## Modes

When you paste code, the skill shows a menu first:

```
Which mode do you want?

/dev        → Developer mode    — intent, decisions, what breaks
/student    → Student mode      — concepts, context, plain English
/interview  → Interview mode    — first-person, 90-second explanation
/smell      → Smell check       — honest assessment of what's wrong
/failures   → Failure modes     — what breaks and what happens when it does
```

You pick the mode. It runs the explanation. No guessing, no defaults.

---

## Mode breakdown

### `/dev` — Developer mode

For developers working on the codebase. Assumes you can read the code — gives you what it doesn't say.

Covers: purpose, behavior, non-obvious decisions, at least one failure path.  
Skips: obvious syntax, generic descriptions, anything already visible in the code.

**Output:**
```
PURPOSE
WHAT IT DOES
FLOW         (only when sequence matters)
KEY DECISIONS
FAILURE MODES
```

---

### `/student` — Student mode

For learners, juniors, or anyone reading a codebase for the first time.

Covers: plain-English explanation, where it fits in the project, the concept it demonstrates, one analogy (only if it helps), what to look at next.  
Skips: deep tradeoff analysis, jargon without explanation, assumed library knowledge.

**Output:**
```
WHAT THIS IS
WHERE IT FITS
THE CONCEPT
FAILURE MODES  (simplified — no stack traces)
WHAT TO LOOK AT NEXT
```

---

### `/interview` — Interview mode

For explaining your own code in a technical interview. First-person, ~200 words, sounds like someone who built it.

Rules: first person throughout, one tradeoff mentioned, under 90 seconds to speak aloud.

**Output:**
```
[natural spoken paragraphs — no headers, no bullets]
FOLLOW-UP READY
```

---

### `/smell` — Smell check

For honest code review. Flags what's wrong and gives a concrete fix for every issue.

Checks: doing too much, missing error handling, invisible dependencies, security holes, logic that breaks, readability, performance.

**Output:**
```
WORKS
REAL PROBLEMS
SMELLS
VERDICT
```

---

### `/failures` — Failure mode coverage

For understanding every way the code can break — useful for debugging, hardening, and interviews.

Separates expected failures (bad input, missing records) from unexpected ones (infrastructure down, env vars missing). Explicitly flags silent failures — the ones that cost you hours with no log and no signal.

**Output:**
```
FAILURE MODES
  Expected:
  Unexpected:
SILENT FAILURES
```

---

## Example — `/dev` on auth middleware

**Input:**
```js
const protect = async (req, res, next) => {
  let token;
  if (req.headers.authorization?.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }
  if (!token) return res.status(401).json({ message: 'Not authorized' });
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    next();
  } catch (err) {
    return res.status(401).json({ message: 'Token invalid or expired' });
  }
};
```

**Output:**
```
PURPOSE
Guards protected routes. Any request without a valid token gets stopped here
before reaching the actual handler.

WHAT IT DOES
Pulls the JWT from the Authorization header, verifies it against the secret,
fetches the matching user from the database, strips the password field, and
attaches the user to req so every downstream handler knows who's making the request.

FLOW
1. Check for Authorization: Bearer <token> header
2. No token → 401, stop
3. Verify token — tampered or expired, jwt.verify throws
4. Fetch user, strip password, attach to req.user
5. Call next() to hand off to the route handler

KEY DECISIONS
Token verified before hitting the database — no point making a round-trip for
something invalid. Password stripped with .select('-password') so it can never
accidentally appear in a downstream response.

The catch block returns the same 401 for both tampered and expired tokens.
Intentional — telling attackers which one failed gives them information.

FAILURE MODES
- No Authorization header → token is undefined → 401 before touching the DB
- Token expired → jwt.verify throws → caught → 401 (same as tampered, on purpose)
- JWT_SECRET missing from env → throws → caught → silent 401. Real error never logged.
- User deleted after token issued → findById returns null → req.user is null →
  downstream handlers crash unless they null-check req.user
```

---

## Example — `/smell` on a raw SQL query

**Input:**
```js
const getUser = async (id) => {
  const user = await db.query(`SELECT * FROM users WHERE id = ${id}`);
  return user;
}
```

**Output:**
```
WORKS
Fetches a user by ID.

REAL PROBLEMS
- SQL injection — id is interpolated directly into the query string.
  Anyone who controls id can run arbitrary SQL on your database.
  Fix: db.query('SELECT * FROM users WHERE id = $1', [id])

SMELLS
- SELECT * fetches every column — password hash, tokens, internal flags.
  Fix: SELECT id, name, email only.
- No null check — if no user exists, caller gets undefined with no signal.
  Fix: return null explicitly or throw a typed NotFoundError.

VERDICT
Not safe to ship. SQL injection is a hard blocker. Fix that first.
```

---

## How to use it

**In Cursor or Windsurf:**  
Add `code-humanizer.md` to your project rules or drop it into your context. Then paste code and type a mode command.

**In Claude or any LLM chat:**  
Paste the contents of `code-humanizer.md` as a system prompt or at the start of your conversation. Then paste your code and pick a mode.

**Trigger phrases** (mode gets detected automatically):
- `"explain this code"` → shows mode menu
- `"/dev"`, `"/student"`, `"/interview"`, `"/smell"`, `"/failures"` → runs that mode directly
- `"explain this as a student"`, `"prep this for an interview"` → auto-detected

---

## What it won't do

- Repeat the code back to you line by line
- Explain obvious things (`"this imports Express"`)
- Use phrases like `"This implementation leverages..."`, `"robust and scalable"`, `"seamlessly integrates"`
- Give you the same five-section template for every piece of code regardless of complexity
- End with a compliment (`"This is a solid implementation"`)


---

## Version

`v1.0.0` — five selectable modes, normalized structure, one worked example per mode, failure mode coverage built into every output.