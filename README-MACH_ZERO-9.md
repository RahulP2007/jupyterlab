# Fixing Duplicate Kernel Sessions in JupyterLab

JupyterLab is an interactive, reproducible computing environment. But it has an annoying quirk: reload a notebook, and a brand-new kernel session quietly starts up while the old one keeps running in the background.

The result? Duplicate outputs, and CPU and RAM being burned on kernels nobody can see or use.

This document explains how we're fixing it, and why we picked this approach over the alternatives.

## The Fix: "Single Path Fixed"

Instead of blindly creating a new session, we make the kernel-selection step check for an existing one first. When a user selects a kernel and the notebook has no attached session, we:

1. Refresh - the session list from the server, so we're not working from stale information.
2. Look for - a session whose path matches this notebook.
3. Attach - to it if one exists.
4. Change - that session's kernel, using the same code path ordinary notebooks already use.

## The Four Options We Considered

### Option A: Attach to the existing session, then change its kernel (chosen)

Before creating anything, the kernel-selection method asks the server whether a session already exists for this notebook's path. If so, it attaches and reuses the existing "change kernel" step.

**Why we like it**

- It fixes the actual cause. The bug happens at creation time, so that's where the check belongs.
- It's small: about 20 added lines in one method, with nothing removed.
- It reuses code that already exists (the session lookup and the "change kernel" branch).
- It keeps the "Notebook (no kernel)" feature working the way it's meant to.

**The catch**
If you pick the same kernel that's already running, the kernel in session A gets replaced with a fresh one, so any running state is lost. That's exactly the question Option B tackles.

---

### Option B: Same as A, but leave the kernel alone if the names match

This is A plus one extra branch. After attaching, if the chosen kernel has the same name as the one already running, we stop there and don't touch it. Only a _different_ kernel name triggers a change.

In our example, you pick Python 3 and kernel A is already Python 3. The notebook attaches to session A and that's it. The loop in kernel A keeps running and nothing is lost.

**Why it's appealing**

- It protects running work in the common case where someone picks the kernel they already have.

**Why we're hesitant**

- It adds a branch, which means more code to review and test.
- It changes what **Select** means. Someone clicking Select on the same kernel might expect a fresh start, and under B nothing visible happens.
- That makes it a product decision rather than a bug fix, so it belongs with the maintainers.

**Verdict:** Our fallback. If the maintainers want it, it's a small addition on top of A.

---

### Option C: Make reload smarter about picking a session

Here we leave the creation logic alone and change the lookup used on open and reload. When several sessions share a path, it picks one by rule (say, the newest, or the one with a running kernel) instead of just taking the first in the list.

In our example, session B still gets created on kernel selection. But after a reload, the notebook picks whichever session the rule prefers, say B, so Restart Kernel hits the kernel actually running the loop.

**What it gets right**

- It would hide the visible symptom in many cases.

**Why it falls short**

- Duplicates still pile up on the server. Session A and its kernel stay alive, eating memory and CPU, and nobody can see them.
- Any rule is a guess. "Newest" or "most active" will be wrong in some scenario, and users end up restarting the wrong kernel again.
- It treats the symptom while leaving the underlying rule (one notebook, one session) broken.

**Verdict:** Not a fix on its own. It could make a decent optional safety net later.

---

### Option D: Remove the "do not reuse" setting from the menu

Delete `shouldReuse: false` from the "Notebook (no kernel)" menu item, so every open attaches to an existing session again.

In our example, the notebook attaches to session A on open, selecting a kernel changes A's kernel, and no duplicate appears.

**What it gets right**

- It's a one-line change, and this exact reproduction goes away.

**Why we rejected it**

- It undoes PR #18856, which exists so people can open a notebook _without_ connecting to an existing kernel. Without it, the menu item no longer does what its label says.
- It hides the bug instead of fixing it. The creation code would still produce duplicates any time a notebook has no attached session but the server has one.

**Verdict:** Rejected. It trades one bug for another and removes a deliberate feature.

---

## Side-by-Side

| Option         | Fixes the cause? | Keeps #18856? | Size of change    | Main risk                                   |
| -------------- | :--------------: | :-----------: | ----------------- | ------------------------------------------- |
| **A** (chosen) |       Yes        |      Yes      | About 20 lines    | Replaces a same-name kernel                 |
| **B**          |       Yes        |      Yes      | A bit more than A | Changes what Select means                   |
| **C**          |        No        |      Yes      | Medium            | Duplicates remain; the rule can guess wrong |
| **D**          |        No        |      No       | One line          | Breaks a deliberate feature                 |

## The Short Version

- **A:** Check first, then reuse.
- **B:** Check first, reuse, and leave a matching kernel alone.
- **C:** Let the duplicates form, then guess better after reload.
- **D:** Remove the feature that exposed the bug.
