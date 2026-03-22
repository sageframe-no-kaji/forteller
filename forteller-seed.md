# Project Seed: Forteller

_Kṣetra-Ops Suite — CLI Deployment Tool_
_Seed created: March 21, 2026_
_Revised: March 21, 2026_

---

## 1. The Problem

Updating a single configuration file on a homelab machine requires an absurd amount of ceremony.

The file lives in a Git repo on your workstation. The machine it belongs on is three feet away, on the same network, reachable by SSH. To get the file from here to there, you have one of four options:

**SSH and edit in place.** Log into the machine, open the file in nano, make the change. The repo is now out of date. You will forget to update it. Weeks later, the repo disagrees with the live machine and you don't know which is correct.

**rsync or scp.** Push the file manually. It arrives. You reload the service by hand. There is no record, no validation, and no memory. Next time, you have to remember the remote path, the service name, and the reload command.

**Ansible.** Write a playbook. Reference an inventory, host variables, roles. Run the playbook. It converges the entire host, including dozens of things you didn't want to touch, to update one file. For a five-machine homelab, the overhead is absurd.

**Mount and drag.** Mount the remote filesystem, navigate, overwrite. Permissions may or may not survive. The service doesn't know the file changed.

Every one of these works. None of them remembers what happened. None of them encodes intent — which file belongs where, what service depends on it, what should happen after it changes. The knowledge of "this file goes to this machine at this path, and afterward you reload this service" lives in the operator's head, and stays there until they forget.

The real-world evidence: the sageframe-config repo restructure required a full working session — SSHing into every machine, running hand-written gather scripts (`kura_gather.sh`, `shingan_gather.sh`), collecting files, mirroring them to a local Git repo. Those gather scripts are literally hand-written Forteller operations. The manifest they produced — which files live on which machines and where they belong — is trivial to encode. Executing it reliably is the hard part.

---

## 2. The Landscape

### chezmoi

The closest analog. Manages dotfiles — `$HOME` configurations on workstations. Vault-first, template-supported, well-designed.

**Where it falls short:** Designed for workstations, not infrastructure. Uses "apply" (pull to host) rather than push. Doesn't handle system-level configs (`/etc/`, `/opt/services/`, systemd units). Doesn't do post-deploy validation or service reload. Doesn't understand the relationship between a config and the service that depends on it. Right tool for the workstation. Wrong tool for the hypervisor.

### Ansible

Industry standard for configuration management. Declarative, idempotent, powerful.

**Where it falls short:** Designed for establishing baseline state across fleets. Driving a truck to deliver a letter. The YAML abstraction layer means the relationship between a config file and its deployment is mediated by playbooks, roles, inventories, and variables — layers of indirection that obscure direct intent. For homelabs with five to fifteen machines, the overhead-to-value ratio is wrong.

Ansible remains the right tool for initial machine setup. Forteller handles the deliberate, incremental changes after the baseline exists.

### GitOps (ArgoCD, Flux)

Reconciliation-loop tools for Kubernetes. Continuously watch a repo and correct drift automatically.

**Where they fall short:** Require agents, assume Kubernetes, and their core mechanic — silent, continuous reconciliation — is exactly what Kṣetra-Ops philosophy rejects. Drift should be observed and resolved by the operator, not automatically corrected.

### Raw scripts (bash + rsync)

The most common actual practice. The gather scripts from the sageframe audit are a perfect example.

**Where they fall short:** No standard structure. No metadata. No memory. Every operator reinvents the same patterns. The scripts accumulate and become their own maintenance burden.

### What's actually missing

A tool that:

- Reads deployment intent from the file itself
- Pushes files via SSH (not pull, not reconciliation)
- Can also gather files from machines into the vault
- Validates before reloading
- Builds the repo tree from the files themselves — the files are the manifest
- Is small enough to understand completely
- Has no agents, no daemons, no YAML sprawl

---

## 3. The Vision

Forteller is a command-line tool for tending the relationship between a vault and the machines it describes.

A config file carries a pragma — a comment block declaring where it belongs and what should happen when it arrives. The file is both the artifact and the manifest. There is no separate playbook. The intent travels with the file.

### Getting started

`fortell init` walks you through setup. Interactive onboarding — generates your Forteller key, sets the vault location, and explains what each step does and why. A new operator goes from install to first deploy in minutes.

### Setup commands

**`fortell init`** — Interactive onboarding. Generates an ed25519 keypair (`~/.forteller/id_forteller`), sets the vault path, walks through first principles. Run once per machine.

**`fortell trust <host>`** — SSH to a machine (using whatever access you have now — password, existing key) and install the Forteller public key into `authorized_keys`. One-time ceremony per host. After this, Forteller uses its own key for everything. No reliance on personal SSH config, no agent forwarding, no ambiguity about which key is doing what.

**`fortell vault <path>`** — Set the vault location. On the workstation: a local path. On field machines: `fortell vault atmarcus@basho:/path/to/repo`. One line of config.

### Core operations

**`fortell deploy <file>`** — Push a file from the vault to the machine it belongs on. The pragma says where. Validates if there's a validation command. Reloads the service if there's a reload command.

**`fortell beam <file>`** — Send a file home from wherever you are in the field. You're logged into a machine, you find a config worth keeping: `fortell beam sanoid.conf`. Forteller reads `$(hostname)` and `$(realpath sanoid.conf)`, writes the pragma header, and sends it to the vault. The repo tree builds itself — `koan/etc/sanoid/sanoid.conf` created automatically because the file came from koan at that path. The file names itself.

**`fortell summon <host> <path>`** — Reach out from the vault and pull a file in. Same result as beam but initiated from the workstation. Replaces hand-written gather scripts.

**`fortell status`** — Scan all files with pragmas, compare against live machines, report drift. No external manifest needed. The files are the manifest.

### Inspection

**`fortell ask <file>`** — Display the parsed pragmas. Where does this file belong? What happens after it arrives? Who does it serve? The Foretellers answer questions. So does Forteller.

**`fortell dry-run <command>`** — Preview any operation. Show what would happen without doing it. Works with deploy, beam, summon.

**`fortell hosts`** — List all machines Forteller has been trusted on.

**`fortell tree`** — Show the vault structure derived from pragmas.

### Quick start (minimum viable ceremony)

```
fortell init                              # generate key, set vault
fortell trust koan                        # install key on first machine
fortell summon koan /etc/sanoid/sanoid.conf  # pull first file into vault
fortell ask sanoid.conf                   # see the pragma it wrote
fortell deploy sanoid.conf                # push it back (round trip proof)
```

Five commands. The vault has its first file, the tree has its first branch, and the operator understands the full cycle.

### Design principles

The repo tree is a derived artifact. You never manually create directories. The tree mirrors the field because the field tells it what exists. Every file is a seed that knows what it is and where it came from.

Forteller manages its own SSH identity. `fortell init` generates the key. `fortell trust` distributes it. No dependence on the operator's personal SSH configuration. The Forteller key is scoped to one purpose: Forteller's operations. This replaces the ad-hoc infra key pattern visible in the audit logs — `id_ed25519_infra` on koan, manually distributed, inconsistently present across machines.

Every command in this vocabulary will be available as a pālana module. The CLI is the API. pālana invokes Forteller as an external process — same commands, same behavior, different surface. What you can do in the terminal, you can do in the workbench.

Full man pages for every command. `fortell help <command>` gives you the full story. The tool that tells should be exemplary in its own documentation.

---

## 4. Audience

**Primary: Me.** The sageframe-config repo manages eleven machines. The manual sync that motivated this project is a real recurring cost. Forteller solves my problem first.

**Secondary: Homelab operators** managing small-to-medium infrastructure (3–20 machines), using Git for config tracking, finding Ansible too heavy for incremental changes. The r/homelab and r/selfhosted communities have a persistent conversation about what sits between chezmoi and Ansible. The answer right now is "scripts." Forteller is a structured answer.

**Tertiary: Small-team DevOps practitioners** managing non-Kubernetes infrastructure who want something lighter than a full configuration management stack.

---

## 5. Identity

**Forteller** — from Ursula K. Le Guin's _The Left Hand of Darkness_. The Foretellers are the Handdara practitioners on Gethen — they answer questions through a collective, ritualized practice of inquiry. The name captures the core mechanic: the file _tells_ the tool what to do. The tool does not infer, guess, or decide. It listens to the file and executes what the file declares.

The naming carries an intentional contrast with the industry's dominant tool: **Ansible**, also from Le Guin's universe — the device that enables instantaneous communication across worlds. Ansible is named after the machine: fast, powerful, impersonal, operating at scale. Forteller is named after the human practice: deliberate, present, ritualized, operating at the scale of a single question and answer. The industry named its tool after the machine. This tool is named after the practice.

Part of the Kṣetra-Ops suite. Governed by bīja. Designed from day one to be invocable by pālana as an external CLI — the first tool in the shed.

---

## 6. Project Nature and Intent

**Open source.** Published on GitHub under `sageframe-no-kaji`.

**Production tool, not a learning vehicle.** The goal is a working tool that solves a real problem.

**Designed for extraction.** No sageframe-specific assumptions in the code. Any Git repo, any SSH-accessible machine, any file with deploy pragmas.

**Designed to be called.** Forteller's CLI is its API. pālana will invoke it as an external process. This is intentional — Unix philosophy. Do one thing. Be callable by others.

**First proof of the Kṣetra-Ops philosophy.** If vault-first, push-only, file-centric deployment is as clear and satisfying as the philosophy claims, it validates the approach before the larger investment in pālana.

**A demonstration of the broader work.** Kṣetra-Ops, bīja, the Ho System, the methodology, the practice. Forteller is a tangible expression of all of it.

---

## 7. Architecture Direction

_Opinions, not commitments._

**Language:** Python. Familiar territory. Rich SSH libraries (Paramiko, Fabric). Reasonable CLI libraries (Typer/Click). pālana's stack is a separate decision — if pālana is built in a different language, Forteller is invoked as a subprocess, which is cleaner than a direct import anyway.

**CLI framework:** Typer (provisional). Clean, type-hinted, generates help text. Built-in support for rich help output that can serve as man pages.

**SSH and identity:** Forteller manages its own ed25519 keypair, generated at `fortell init` and stored in `~/.forteller/`. The `fortell trust <host>` ceremony installs the public key on a machine using whatever access the operator currently has. After trust is established, all Forteller operations use the Forteller key exclusively. This eliminates dependence on personal SSH config, agent forwarding, or the ad-hoc infra key patterns visible in the audit logs. Paramiko or Fabric for transport.

**Deploy pragma format:**

```conf
# @fortell host=koan path=/etc/sanoid/sanoid.conf
# @fortell validate="sanoid --configcheck"
# @fortell reload="systemctl restart sanoid"
```

One directive per line. Key=value pairs. Comments in any supported style (`#`, `//`, `;`). Ignorable by every runtime. Trivial to write. Parseable with regex.

**Beam auto-pragma:** When beaming a file, Forteller generates the pragma from `$(hostname)` and `$(realpath file)`. The operator can edit it later. The initial pragma is always correct because it describes where the file actually was.

**Tree generation:** The repo tree is built from pragmas. `host=koan path=/etc/sanoid/sanoid.conf` → file stored at `koan/etc/sanoid/sanoid.conf` in the vault. Directories created automatically. The tree is never manually maintained.

**Status via scanning:** `fortell status` walks the repo, finds files with `@fortell` pragmas, SSHes to each host, diffs vault against live. No external manifest. The files are the manifest.

**Onboarding:** `fortell init` is interactive. It generates the key, asks for the vault path, explains each step, and optionally walks through trusting the first host and summoning the first file. The goal: install to first deploy in minutes.

**No state file. No manifest file. No daemon.** Git is the memory. The pragmas are the manifest. Forteller runs when invoked and exits.

**Install everywhere.** `pip install forteller` or a single-file script. On field machines, one config line for the vault address. Like Git — you install it because you want the vocabulary available wherever you're standing.

**Man pages.** Every command has full documentation accessible via `fortell help <command>`. The tool that tells must be exemplary in its own documentation.

---

## 8. Constraints

**Skills:** Not a developer. Build production software using the Ho System with AI collaboration. Shipped Kanyō (Python, 278 tests), Hōzō (Python, containerized), m4Bookmaker (Python/PyQt). Python CLI tools are proven territory.

**Time:** Buildable in a weekend. The scope is deliberately tight and the problem is well understood.

**Infrastructure:** Must work with the existing sageframe topology — mixed bare metal, LXC, and VM hosts, all reachable via SSH. Must also work for any homelab operator with SSH access to their machines.

**Security:** Forteller generates and manages its own SSH key. The key must be ed25519. Private key never leaves `~/.forteller/`. Trust ceremony must work with password auth or existing keys as a bootstrap.

**pālana compatibility:** Forteller is invoked as an external CLI. Clean stdin/stdout interface. Exit codes for success/failure. Structured output (JSON flag) for machine consumption. The CLI is the API.

**No sageframe-specific assumptions.** The tool works with any Git repo and any SSH-accessible machines.

---

## 9. Scope Boundaries

**Forteller IS:**

- A CLI for tending the relationship between a vault and its machines
- Self-keyed (manages its own SSH identity via init/trust)
- Three core operations: deploy (push), beam (send home), summon (pull in)
- Self-inspecting: ask, hosts, tree, status
- Deploy-pragma-driven (intent embedded in the file)
- Self-building (repo tree generated from pragmas, never maintained by hand)
- Capable of auto-generating pragmas on beam/summon
- Capable of validation and service reload after deploy
- Capable of drift detection via status
- Installable on any machine in the field
- Fully documented with man pages for every command
- Interactive onboarding via `fortell init`
- Designed to be invoked by pālana — every CLI command is a pālana operation

**Forteller is NOT:**

- A file manager (that's pālana)
- A configuration management system (that's Ansible)
- A secrets manager
- A Git tool (reads from repo, doesn't commit/push/pull)
- A daemon or agent
- A monitoring tool
- Safe-by-default (assumes the operator understands consequences)

**Full command set:**

Setup: `init`, `trust`, `vault`
Operations: `deploy`, `beam`, `summon`, `status`
Inspection: `ask`, `dry-run`, `hosts`, `tree`
Help: `help <command>` (man pages)

**Later:**

- Multiple hosts per pragma
- Pre/post hooks
- Checksums
- `--json` output flag for pālana consumption
- Editor integration (VS Code command)
- Confirmation prompts (optional)

---

## 10. Success Criteria

1. **Single-command config update.** Edit a file in the repo, `fortell deploy` it, service is running the new config.

2. **Single-command file capture.** Standing on any machine, `fortell beam sanoid.conf` sends it to the vault with correct pragma and tree placement.

3. **Self-building tree.** The repo directory structure is generated entirely from pragmas. No manual directory creation, ever.

4. **Drift visibility.** `fortell status` tells me which files on which machines have diverged from the vault.

5. **Obvious pragmas.** Someone opening a config file for the first time can read the pragma block and understand where the file goes and what happens after, without documentation.

6. **Self-keyed.** `fortell init` + `fortell trust <host>` and the SSH identity is handled. No manual key distribution, no agent forwarding, no ambiguity.

7. **Minutes to first deploy.** A new operator runs `fortell init`, trusts a host, summons a file, deploys it back. Five commands. The full cycle is proven.

8. **Ask tells the truth.** `fortell ask <file>` displays everything Forteller knows about a file — where it goes, what it does after, when it was last deployed. The tool that tells must be able to answer when asked.

9. **Loud failure.** SSH unreachable → clear error. Validation fails → clear error with output. No pragmas → clear error. No silent failures.

10. **Small enough to understand.** A competent Python developer reads the entire codebase in an afternoon.

11. **pālana-ready.** Every command works identically when invoked by pālana as a subprocess. The CLI is the API.

---

## 11. Where I'm Starting From

**Strong territory:**

- Multiple production Python CLI tools shipped
- Ho System methodology — will use it
- Know the problem space intimately — have been doing this manually for years
- Working SSH infrastructure and Git repo of config files ready
- The gather scripts in the audit logs are literally hand-written Forteller operations

**Familiar but not deep:**

- Paramiko/Fabric for SSH transport
- CLI framework design (Typer/Click)
- rsync programmatic invocation

**New territory:**

- Publishing a CLI tool for external use (packaging, pip install, docs for strangers)
- Designing a CLI that another application (pālana) will invoke programmatically
- Auto-generating repo structure from file metadata

---

## 12. What I Want to Learn

**Primary:** Whether the bīja philosophy works in practice — whether vault-first, pragma-driven deployment is as clear and satisfying in use as it is in theory.

**Secondary:** How to design a CLI that is simultaneously a human interface and a machine interface (for pālana).

**Tertiary:** Whether the self-manifesting tree — repo structure generated from pragmas — is as powerful as it feels conceptually.

---

## 13. Open Questions

**Pragma prefix:** `@fortell` or `@deploy` or `@bija`? `@fortell` ties to the tool. `@deploy` is generic. `@bija` ties to the philosophy. Provisional: `@fortell` — the tool's name is the namespace.

**Pragma location in file:** Top of file only. Simpler to parse, more visible to humans.

**Git integration on beam/summon:** After beaming or summoning, the file is in the vault but not committed. Should Forteller auto-commit? Provisional: no. Forteller moves files. Git commits are the operator's responsibility. Separation of concerns.

**Conflict on beam/summon:** What if the file already exists in the vault? Overwrite? Diff and ask? Provisional: show diff, require `--force` to overwrite.

**Files without comment syntax:** Binary files, JSON (no comments). How are pragmas stored? Provisional: sidecar `.fortell` file — `config.json.fortell` — same pragma format, separate file.

**Trust ceremony details:** `fortell trust <host>` needs to SSH in using whatever access currently works. Password? Existing key? Should it try the operator's default SSH config first and fall back to password prompt? Provisional: try SSH config first, prompt for password if that fails. The goal is one successful connection to install the Forteller key, after which the key handles everything.

**Key revocation:** If the Forteller key is compromised, how do you revoke it across all trusted hosts? `fortell untrust <host>` to remove the key from one machine? `fortell untrust --all`? Provisional: build untrust. It's the responsible complement to trust.

**Zencat (router):** Merlin doesn't run Python. Beam won't work from zencat. Summon from the workstation works fine. Acceptable asymmetry — some machines are beam-capable, some are summon-only. Trust should still work (it only needs SSH access, not a local Forteller install).

**Status performance:** Scanning every file and SSHing to every host could be slow for large repos. Provisional: solve it when it's a problem. Parallel SSH is the obvious optimization.

---

## The Soul and the Body

**Soul:** Every file knows where it belongs. A single command makes it so — in either direction. And if you ask, it tells you.

**Body:** A Python CLI that manages its own SSH identity, reads deploy pragmas from config files, pushes files to machines, pulls files into a self-building vault, reports drift, and answers when asked.

---

## Context Documents

- `Kṣetra-Ops Philosophy Brief` — governing philosophy
- `forteller.md` — original spec and design
- `real-world-context.md` — actual pain points and topology
- `repo-config-plan.md` — the manual process Forteller replaces
- `*_audit.txt` / `*_gather.sh` — the hand-written gather operations that are literally Forteller's specification
- `bija-human-based-git-ops.md` — earlier iteration that became Forteller
- `Ksetra-Ops-README.md` — suite philosophy
- `github.com/bahamas10/sshp` — parallel SSH executor. Different problem (breadth: one command across many hosts) but useful reference for `fortell status` pattern: parallel SSH with aggregated output, join mode that groups hosts by identical results. Relevant for drift reporting.
