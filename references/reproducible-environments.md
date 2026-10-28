# Reproducible environments

*October 2026*

A program's output depends on more than its code. This piece names everything else it depends on, shows how to pin each part, and ends with a test you can run against your own project in five minutes.

## Two laptops, one commit

You and a colleague check out the same commit. A script sorts five filenames. Your output and theirs differ, and nobody changed a line. Try it, on any Linux machine with glibc:

```
printf 'apple\nBanana\n_config\ncherry\nZebra\n' > names.txt
LC_ALL=C sort names.txt
LC_ALL=en_US.UTF-8 sort names.txt
```

| `LC_ALL=C` | `LC_ALL=en_US.UTF-8` |
| --- | --- |
| Banana | apple |
| Zebra | Banana |
| \_config | cherry |
| apple | \_config |
| cherry | Zebra |

The C locale sorts by byte: capitals first, then the underscore, then lower case. The English locale sorts the way a dictionary does: case folded, punctuation mostly ignored. Neither is wrong. The script never said which one it wanted, so it got whichever the machine had set in `LANG`. A fresh Docker image usually has none (so C); a desktop usually has `en_US.UTF-8`. macOS ships its own collation tables, a third possibility.

If that sorted list feeds a snapshot test, a generated file or a cache key, the two laptops now disagree about whether the build is correct. Nothing in the repository explains why. That is the whole subject in one terminal: **the locale was an input to the program, and nobody declared it.**

## Hidden inputs

An environment is everything your program depends on besides its own code. Eelco Dolstra, who designed Nix, put the consequence plainly: treat a build as a function of its inputs, and every "works on my machine" bug is an input you forgot to list.

A **reproducible environment** declares those inputs, so nothing reaches in from the machine unannounced. Declaring has three parts:

1. **Name** the input somewhere in the repository (a version, a hash, a URL).
2. **Pin** it to one exact value, so "Python 3" becomes "Python 3.12.7".
3. **Fence** the program off from everything you didn't name, so an undeclared input fails loudly instead of leaking in quietly.

The third part is the one people skip. A lockfile names your packages, but if the script still picks up whatever `sort` or `ffmpeg` is first on your `PATH`, that tool is a hidden input with a lockfile standing next to it.

The reproducible-builds community adds a harder bar: reproducible means someone else can rebuild it and check the result, not that it worked for you twice. Keep both ideas in hand. Every section below asks the same question of one layer: what reaches in from the machine here, and how do you make it declared?

## The stack

Here is every input of a typical project, as layers. The rest of this piece walks down the stack and ends with this exact picture: the six upper layers declared in the repository, and the two below the line deliberately left to the machine.

```
Layer                              Declared by                                Status
+--------------------------------------------------------------------------------------+
| 1  Code and content              the commit                                 pinned   |
| 2  Language packages             a lockfile with hashes                     pinned   |
| 3  Language runtimes             a version file                             pinned   |
| 4  System tools                  a Nix flake or an image digest             pinned   |
| 5  Downloaded assets             a URL plus a checksum or commit            pinned   |
| 6  OS, CPU and process settings  explicit settings, a CI job per platform   narrowed |
+--------------------------------------------------------------------------------------+
================ the line: nothing below it belongs in a package =======================
+ - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -+
| 7  Secrets                       one decryption key per machine        provided once |
| 8  Network permission            the machine owner's allowlist         provided once |
+ - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -+
```

Layer 6 says "narrowed" rather than "pinned" because no repository can choose your CPU; the section on it explains what you can do instead.

## The six layers you can pin

Each upper layer has a standard way to pin it, and each pin has a price. The table is the reference; the notes under it are the parts people get wrong.

| Layer | What drifts if you don't pin it | How people pin it | What the pin costs |
| --- | --- | --- | --- |
| 1. Code and content | A file the build reads but git ignores: a local config, a generated file, a dataset in `~/Downloads`. | Commit it, or generate it from something committed. The commit hash names the layer. | Discipline, and repository size for big files. |
| 2. Language packages | `requests>=2` resolves to whatever is newest today; a dependency of a dependency ships a breaking release. | A lockfile with exact versions and hashes: `package-lock.json`, `Cargo.lock`, `uv.lock`, and since 2025 Python's standard `pylock.toml`. | Updating becomes a chore you schedule, and lockfile diffs in review. |
| 3. Language runtimes | Python 3.12 removed `distutils`, so code that ran on 3.11 fails to import. | A version file (`.python-version`, `.nvmrc`, `.tool-versions`) plus a tool that installs exactly that version. | Every machine needs the installer (uv, nvm, mise, pyenv). |
| 4. System tools | `sed -i` takes no argument in GNU sed and requires one in BSD sed, so a script that works on Linux fails on a Mac. An `ffmpeg` upgrade changes encoder defaults. | A Nix flake with its `flake.lock`, or a container image referenced by digest (`image@sha256:...`), not by tag. | A real learning curve (Nix) or a VM and gigabytes of images (containers). |
| 5. Downloaded assets | Model weights fetched from a branch named `main` change under you; a missing font makes the renderer silently fall back to another. | A URL plus a checksum, or a URL at a fixed commit. Verify the hash after every download. | Storage, and the host has to stay up: mirror what you can't lose. |
| 6. OS, CPU and process settings | Locale, timezone, `PATH` order, an environment variable you set months ago; ARM and x86 builds of a numeric library rounding differently. | Set process settings explicitly in the script (`LC_ALL`, `TZ`, a seed). For the platform: name the ones you support and test each one in CI. | A CI matrix. You can narrow this layer but never fully pin it. |

**Fence, not just pin.** A pin only helps if the program can't reach around it. Run scripts with an explicit `PATH` or inside the pinned shell, pass `LC_ALL=C` and `TZ=UTC` where output order or times matter, and make the test suite fail when a tool is missing rather than quietly using the system's copy.

**Layer 6 is where honesty starts.** Containers fix the userland (layers 3 to 5) but share the host's kernel and CPU. On a Mac, a Linux container runs inside a virtual machine, so you are testing the VM, not the Mac. Treat each platform as a separate claim that needs its own test.

**Nix or containers?** Both pin layer 4. A container ships a whole Linux filesystem, frozen; it is simple to explain and runs the same everywhere it runs, which on a Mac means in a VM. Nix builds each pinned package natively for each platform from one lockfile, so a Mac and a Linux box get the same versions without a VM, at the cost of a steep learning curve (and, as of late 2026, flakes still sit behind an experimental flag). Dev Containers and Codespaces wrap a container plus editor settings into one `devcontainer.json`, which is the gentlest way to hand a newcomer the whole setup. Pick by who runs the code: all-Linux teams and CI do fine with containers; mixed Mac and Linux teams who want native speed tend to end up at Nix.

## Steps that can't be pinned: save their output instead

Some steps are impure by nature. A hosted AI model, a web API, a price feed, anything random: you can pin the request, but not the answer. The reproducible move is to run the step once, save the output under a hash of everything that went in, and commit it.

```
key  = sha256(model + voice + settings + exact input text)
path = cache/<key>.mp3
if path exists: use it
else: call the API once, write path, commit it
```

The cache is now an input like any other, the same way a lockfile entry is. A fresh clone rebuilds the identical output without calling anyone, and only changed inputs trigger new calls. Two details matter. Key on content, never on file times: Linux timestamps tick coarsely, and a fresh clone resets them all. Include every setting in the key, so changing the voice or model can't serve a stale file.

**Why a seed isn't enough.** Many APIs take a `seed` parameter, and the providers themselves say not to rely on it. ElevenLabs' text-to-speech reference says a seed makes the system "make a best effort to sample deterministically", with repeated requests usually but not always matching. OpenAI's guide says "determinism is not guaranteed" and points to a `system_fingerprint` that changes when they change their systems. A hosted model is also replaced over time, and nothing in your repository can pin a server you don't run. A seed reduces variation today; a committed output removes it.

## Below the line: what no package should carry

Two layers stay open on purpose, and that is correct, not a gap.

- **Secrets.** Anything everyone can copy can't hold a secret. A package, an image or a repository is built to be copied.
- **Network permission.** Whether a machine may reach a host is the machine owner's decision: a company firewall, a sandbox, a CI runner's allowlist. A package can name the hosts it needs; it can't grant itself access.

So the goal for these two is not "pinned" but "declared, and provided by the machine once".

**The open-source test.** Picture making your repository public, right now, with its full history. What leaks? Adam Wiggins' Twelve-Factor App uses exactly this as the test for whether config is separated from code. If the answer is "nothing", the code layer is clean. Keep the question in mind; it comes back in the next section with a twist.

**Handling secrets well** comes down to a few habits, most of them from OWASP's Secrets Management Cheat Sheet:

1. **Out of the code.** Nothing secret in the repository, the image or the build arguments.
2. **Few copies.** Every copy (a dotfile, a password manager, a CI setting) is one more place to leak and one more place to update.
3. **Small keys.** Scope each key to its one job and cap what it can spend, so a leak costs little.
4. **Read at use.** Fetch the secret inside the process that needs it, not `export`ed into every shell. OWASP notes environment variables "are generally accessible to all processes and may be included in logs or system dumps".
5. **Cheap rotation.** If replacing a key takes one command, you'll do it when you should.
6. **Never printed.** Tools report a secret as `present` or `absent`, never its value or its first few characters.

**The nearest thing to packaging a secret** is committing it encrypted. Tools such as sops (with age keys), git-crypt and agenix keep an encrypted file in the repository; each machine holds one private key that decrypts all of them:

```
age-keygen -o key.txt                               # one per machine; prints its public key
sops encrypt --age age1...,age1... secrets.yaml > secrets.enc.yaml
```

Sops then finds the private key in a standard file (`~/.config/sops/age/keys.txt` on Linux) or in `SOPS_AGE_KEY_FILE` or `SOPS_AGE_KEY`. Be clear about the residue. That one private key per machine is the irreducible setup step: something has to be handed over by a person or a platform's secret store. And removing a machine means more than deleting its key, because it already saw every secret: you rotate each one.

## AI agents turn the environment into a log

Run the open-source test again, with a twist. You work with an AI coding agent. Picture its transcripts made public too: every command it ran, every output it read. What leaks now?

This is the new part. An agent runs commands on your behalf and reads their output, and that conversation is stored: on disk, in a service's history, sometimes shared with a colleague or pasted into a bug report. Anything printed once is kept. Your environment used to be invisible state; with an agent in the loop, it becomes a log.

Here are five ordinary ways a key ends up in that log. For each, asking the agent to be careful is not a defence: it is debugging under pressure, and so are you. The defence is to remove the thing it could leak.

| The agent... | What lands in the transcript | The defence that removes it |
| --- | --- | --- |
| runs `env` to debug a missing variable | Every exported variable, keys included | Keep secrets out of the shell environment. The program that needs a key decrypts it in its own process, so `env` has nothing to show. |
| writes `API_KEY=sk-... ./run.sh` inline | The key, in the command itself (and in shell history) | No command takes a secret as an argument or assignment. Tools fetch their own key, so the agent never holds a value to type. |
| runs a script with `set -x` | Every expanded command, including the one that sent the key | Don't let the shell expand secrets at all: the final program reads the key itself, so the trace only ever shows the program's name. |
| runs a test that fails and dumps its environment or locals | Whatever the process had in memory | Tests run with fake keys by default; the few that need a real one are opt-in. Wrap keys in a type that prints as `**********` (pydantic's `SecretStr` does this). |
| runs `cat .env` to "check the config" | The whole file | Have no plaintext secrets file. What's on disk is encrypted, so `cat` shows ciphertext; config is checked by a doctor that prints `present` or `absent`. |

Notice the pattern: every defence is one of the habits from the previous section (read at use, never printed, few copies) made strict enough that a well-meaning agent can't get around it. An agent is just the most thorough reader of your environment you have ever had.

## How you know

A claim of reproducibility needs a test that would fail if the claim were false. Your own laptop can't be that test: it is the machine most full of hidden inputs. Four checks cover it.

1. **A clean-room run.** Clone the commit into an empty container that has nothing of yours in it, run the bootstrap script, run the tests. Anything it needs that isn't declared fails here first. Run it after adding any dependency, and once with the network off if you claim offline builds.
2. **CI on every push, plus a weekly run.** Every push proves the pins still work. The weekly run on an unchanged commit catches drift upstream of your pins: a download URL that died, a package removed from a registry, a base image deleted.
3. **A doctor.** One command that checks each layer and prints one line per problem, with the fix: `ffmpeg: missing (run script/bootstrap)`. When everything is fine it prints a single green line. It is also how a secret is checked without printing it.
4. **A receipt.** One command that prints what this machine actually runs with, a line per tool: `python 3.12.7`, `node 22.11.0`, `ffmpeg 7.1`, `lockfile sha256:3f9a...`. Save one from each machine, and when two machines disagree, `diff` the receipts before you debug anything else.

The doctor tells you what's missing. The receipt tells you what's different. Between them, "works on my machine" turns into a line you can point at.

## How far to go

Most projects need a lockfile, a version file and a README, not Nix. Climb a rung when the one below stops paying for itself.

| Rung | What you add | Who needs it |
| --- | --- | --- |
| 1 | A README that names the versions and the setup steps | Everyone, including a weekend script you'll run again in a year |
| 2 | Lockfiles with hashes, and a runtime version file | Anything with dependencies that more than one person or machine runs |
| 3 | A pinned toolchain: a Nix flake or a dev container | Projects that call system tools (ffmpeg, a browser, a compiler) or run on both Mac and Linux |
| 4 | Clean-room CI, a doctor, a receipt, encrypted secrets | Projects run by agents or in cloud sessions, or whose output must match byte for byte across machines |

The rungs are cumulative, and rung 2 does most of the work for most code. Rung 4 earns its cost when the machine running the code isn't one you set up by hand.

## What you still trust

Fully reproducible never means trusting nothing. Ken Thompson's "Reflections on Trusting Trust" showed that a compiler can insert code that appears in no source you can read; every chain of checks ends somewhere you didn't check. In practice the chain ends at the Nix binary cache or the registry that serves your base image, the compiler that built them, the package registry behind your lockfile hashes, and the hardware. Pinning makes those trust points few, named and the same on every machine. That is the realistic goal: know exactly where your trust sits, rather than pretend it isn't there.

## A worked example

Here is the layer-by-layer test applied to one real project: papershop, a toolkit that renders narrated explainer videos.

| Layer | Status | How |
| --- | --- | --- |
| Code and content | Pinned | Everything the render reads is committed |
| Language packages | Pinned | `uv.lock` for Python, `package-lock.json` for Node |
| Runtimes | Pinned | `.python-version`, with uv and Python fixed by the Nix lockfile |
| System tools | Pinned | ffmpeg, sox and Node from a Nix flake |
| Downloaded assets | Pinned | Local speech model weights at a fixed commit; hosted narration cached by content hash and committed |
| OS and CPU | Leaking | Mac and Linux frames differ from the layout step on, probably fonts: an open, named gap |
| Secrets | Provided once | One narration API key, moving into an encrypted file (sops and age) so each machine needs only its own age key |
| Network | Provided once | Two hosts (the model download and the speech API) must be allowed; cloud sandboxes block both by default |

The one leaking layer is known and written down, which is the point: the test is not "is everything pinned" but "do you know which parts aren't".

## The fresh-machine ritual

When the layers are in place, a new laptop, a CI runner or an agent's cloud sandbox gets ready in five steps:

1. **Clone** the repository.
2. **Bootstrap**: one idempotent script installs every pinned layer.
3. **Give it one key**: the machine's decryption key, from wherever you keep it.
4. **Allow the hosts** the doctor names, if the network is fenced.
5. **Check the doctor** is green.

The first time, the doctor reads like a to-do list:

```
$ script/doctor
not ready: ffmpeg missing (run script/bootstrap) · secrets: no age key (see secrets/README) · api.example.com unreachable (allow it in the sandbox's network settings)
```

Two steps later:

```
$ script/doctor
ready: tools, packages, assets, secrets present, network ok
```

That second line is the whole goal of this piece, and it is reachable with ordinary tools in a week of small changes.

## Hidden inputs checklist

Ten questions, each naming an input that commonly reaches in from the machine. Run them against your own project; every box you can't tick is a line for your README, your lockfile or your doctor.

- [ ] **Locale.** Does any output depend on sort order, number or date formatting? Is `LC_ALL` set where it matters?
- [ ] **Timezone.** Do timestamps or "today" come from the machine's zone instead of an explicit `TZ` or UTC?
- [ ] **Clock.** Does output embed the current time, so two runs a second apart differ?
- [ ] **Random seed.** Is everything random seeded, and is the seed written down?
- [ ] **CPU architecture.** Have you run it on both ARM and x86, or said which one you support?
- [ ] **Fonts.** Does anything render text? Are the fonts shipped with the project or left to the system?
- [ ] **PATH order.** Could a different `python`, `sed` or `ffmpeg` come first on another machine?
- [ ] **Environment variables.** Does the code read a variable that only your shell sets?
- [ ] **Network hosts.** Which hosts does it reach, and is the list written down?
- [ ] **Downloads.** Is every downloaded file pinned by a checksum or a commit, not a branch or "latest"?

## Sources

Pages opened on 2026-10-28. The opening's `sort` output was run on Ubuntu with glibc 2.39.

- [Reproducible Builds: definition](https://reproducible-builds.org/docs/definition/): "any party can recreate bit-by-bit identical copies"
- [The Twelve-Factor App: III. Config](https://12factor.net/config): the open-source litmus test
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html): environment variables, logging, rotation
- [PEP 751](https://peps.python.org/pep-751/): `pylock.toml`, accepted 31 March 2025, status Final
- [nix.dev: Flakes](https://nix.dev/concepts/flakes): "an experimental extension format", enabled with `--experimental-features 'nix-command flakes'`
- [GitHub Docs: Codespaces secrets](https://docs.github.com/en/codespaces/managing-your-codespaces/managing-your-account-specific-secrets-for-github-codespaces): secrets are exported as environment variables into the codespace
- [GitHub Docs: recommended secrets in devcontainer.json](https://docs.github.com/en/enterprise-cloud@latest/codespaces/setting-up-your-project-for-codespaces/configuring-dev-containers/specifying-recommended-secrets-for-a-repository): the file names secrets, never their values
- [SOPS: age](https://getsops.io/docs/usage/identities/age/): `sops encrypt --age`, key file locations, `SOPS_AGE_KEY_FILE`
- [age](https://github.com/FiloSottile/age): `age-keygen -o key.txt`
- [ElevenLabs API: text to speech](https://elevenlabs.io/docs/api-reference/text-to-speech/convert): `seed` is a "best effort"
- [OpenAI Cookbook: reproducible outputs with seed](https://developers.openai.com/cookbook/examples/reproducible_outputs_with_the_seed_parameter): "determinism is not guaranteed", `system_fingerprint`
- [Ken Thompson, Reflections on Trusting Trust (1984)](https://www.cl.cam.ac.uk/teaching/2223/R209/Reflections-Trusting-Trust.pdf)
