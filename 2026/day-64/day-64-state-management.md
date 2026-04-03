# Day 64 — state notes (Terraform)

Stuff I’m keeping straight for remote state, locking, imports, and not shooting myself in the foot in prod.

---

## State locking

Only one thing should write state at a time. Otherwise two applies can stomp each other and your state file in S3 ends up wrong.

This repo uses **S3** + **DynamoDB** for locks (`terraweek-state-lock` in `terraform.tf`). Plan/apply grabs a lock row, finishes, releases it.

### The annoying error

Usually something like:

```text
Error: Error acquiring the state lock
...
Terraform acquires a state lock to protect the state from being written
by multiple users at the same time. Please resolve the issue above and try
again.
```

Might also see “lock already held”, or DynamoDB/S3 noise if IAM/backend is misconfigured.

If it’s stuck: someone else is running terraform, or a run died and left a stale lock. Terraform prints a **lock ID** — you only need it for `terraform force-unlock <id>`.

**Don’t** force-unlock if a teammate or CI is actually mid-apply. Only when you’re sure nothing is using that state.

---

## Why locking matters with a team

Everyone points at the same state blob (here: S3). No lock → two people both think they’re at version T, both apply, last write wins, state drifts from reality, weird destroys, duplicate resources, bad day.

Lock = one writer at a time. It’s not magic (you still need to coordinate), but it stops silent corruption from parallel applies.

---

## `import` vs creating from scratch

| | apply (new resource) | import |
|---|----------------------|--------|
| Thing in cloud | doesn’t exist yet; terraform creates it | already exists |
| What changes | API create + state update | basically just state (links existing ID to an address) |
| Your job | write HCL, plan, apply | write HCL that **matches** reality, then import; then fix plan until it’s sane |

**Create from scratch:** normal path. You define `resource`, apply, terraform creates it and tracks it.

**Import:** you already have the thing (clickops, old stack, whatever). `terraform import <address> <id>` does **not** write your `.tf` for you — it only wires state. You still need the right `resource` block or the next plan will look insane.

Rule of thumb: scratch = terraform authored the thing. Import = you’re telling terraform “this thing is mine now” and then you make code match.

---

## `state mv` vs `state rm`

Neither edits your `.tf` files. Both are state surgery — run `plan` after.

**`terraform state mv OLD NEW`** — same real resource, new address in state. Use when you renamed a resource or moved it into/out of a module and you don’t want terraform to destroy+recreate. Terraform 1.1+ has `moved` blocks in code if you want that in Git instead of a one-off CLI move.

**`terraform state rm ADDRESS`** — drops that item from state. Does **not** delete the cloud resource by itself. Use to “forget” something (often after removing it from code) or clean up garbage state / re-import elsewhere.

Gotcha with `rm`: if the resource still exists in AWS and is still in your code, next apply might try to create a **second** one. If it’s gone from code but still in the account, you’ve got an orphan. Always `plan` after.

---

## Drift in prod (how people actually reduce it)

**Drift** = AWS (or whatever) doesn’t match what terraform thinks — console edits, CLI, another tool changed tags, sizes, etc. Next `plan` surprises you.

Rough playbook:

1. **Don’t let everyone edit prod in the console.** Read-only or tight roles for most people; break-glass admin when something’s on fire. If every ticket is fixed with a console click, terraform will always lose.

2. **Ship infra changes through CI/CD** — merge to main, pipeline runs plan/apply with a **service role**, not random laptop applies to prod. PRs get a plan so you see weirdness before merge.

3. **Optional:** nightly or scheduled `plan` in prod to page when drift shows up.

4. **Emergency console fix?** Cool, but then update the terraform (and import if needed) so the next apply doesn’t undo you or fight reality.


---

That’s it. When in doubt: `terraform plan`, don’t unlock blindly, and don’t `state rm` without knowing what’s still in code vs what’s still in the cloud.
