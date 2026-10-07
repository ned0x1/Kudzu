# kudzu

> Turn the passwords you find during an audit into the passwords you'll find next.

![version](https://img.shields.io/badge/version-0.1.0-blue)
![license](https://img.shields.io/badge/license-MIT-green)
![python](https://img.shields.io/badge/python-3.10%2B-blue)

## Description

**kudzu** is a command-line tool for pentesters. It capitalizes on the passwords discovered mid-engagement — a Kerberoasted service account, a password dumped from a mailbox, a hit from a previous spray — and automatically expands them into a **targeted wordlist** for further password spraying or brute forcing, on any kind of audit: internal (Active Directory — LDAP, SMB, Kerberos, RDP...), external, web applications, cloud, mail, VPN, whatever the engagement touches.

The name is a nod to the kudzu vine: an invasive, fast-spreading climbing plant — much like the variants of a single leaked password, which tend to proliferate across an environment from one seed (`Company2022` → `Company2023`...`Company2028`, `Company_2023`, `C0mpany2023`...).

kudzu is deliberately small and scriptable. It does one job — grow a seed list into a mutated candidate list — so you don't have to reinvent a rule engine every engagement.

## Features

- **`add`** — store passwords found during an audit, with optional context tags (`--tag smb`, `--tag "mail amy.hopkins"`) and an explicit, per-password choice of whether to mutate it (`--rules`)
- **`create-list`** — generate a wordlist from everything stored: passwords added with `--rules` are expanded through a small, deterministic mutation engine; passwords added without it are kept as-is
- **`list`** — inspect everything stored, including each entry's tag and rules status
- **`remove`** — drop a single password (`--password`) or every entry under a tag (`--tag`)
- **`import`** — bulk-import passwords from an existing file (`secretsdump` output, a cracked-hash list, etc.)
- Configurable via `~/.kudzu/config.yaml` — enable/disable individual rules, tune the year range, digit spread, symbols, leet mapping
- Optional **GPG symmetric encryption** of the local password store

## Installation

### From PyPI

```bash
pipx install kudzu
# or
pip install kudzu
```

### From source

```bash
git clone https://github.com/ned0x1/kudzu.git
cd kudzu
pip install -e .
```

### Requirements

- Python 3.10+
- Linux (Kali, Parrot) — primary target; macOS works too
- `gpg` on PATH, only if you enable encrypted storage

## Quick Start

```bash
kudzu add "Shinra2022" --tag kerberoast --rules
kudzu add "eqYQ5_RXZtUYPJ" --tag "mail ashleigh.lewis"   # random password manager output, no --rules: kept as-is, not mutated
kudzu create-list wordlist.txt --keyword Shinra
netexec smb 10.10.10.0/24 -u users.txt -p wordlist.txt --continue-on-success
```

That's the whole workflow: **add → create-list → spray**.

## Detailed Usage

### `kudzu add <password>`

Adds a password to the local store at `~/.kudzu/password/list.txt` (created automatically). Duplicates are silently skipped.

```bash
kudzu add "Welcome2025" --tag ldap --rules
```

| Option | Description |
|---|---|
| `--tag <tag>` | Context tag to trace where the password came from (e.g. `smb`, `ldap`, `kerberoast`, `mail victor.davis`) |
| `--rules` | Apply the mutation rules to this password in `create-list`. **Without this flag the password is still stored and still included as-is in the generated wordlist, but nothing is built around it.** Use it for passwords with a human-guessable pattern; skip it for high-entropy / randomly generated credentials (password managers, strict policies) where mutating is pointless. |
| `-v`, `--verbose` | Print what the command is doing |

### `kudzu create-list <output_path>`

Reads every stored password. Entries added with `--rules` go through the mutation engine; entries added without it are written out unchanged. Output is deduplicated, one candidate per line.

```bash
kudzu create-list sprayable.txt --keyword Shinra
```

| Option | Description |
|---|---|
| `--keyword <word>` | Extra word to combine into tested passwords — a company name, target name, project codename (`Shinra2026!`, `Shinra_2026`...) |
| `-v`, `--verbose` | Show how many variants each seed password produced, or that it was kept as-is |

### `kudzu list`

Prints every stored password with its tag, timestamp, and whether rules are on for it.

### `kudzu remove`

Removes either one exact password, or every entry sharing a tag.

```bash
kudzu remove --password "Welcome2025"
kudzu remove --tag "mail amy.hopkins"
```

Exactly one of `--password` / `--tag` is required.

### `kudzu clear`

```bash
kudzu clear --yes   # wipe the whole store, skip the confirmation prompt
```

### `kudzu import <file>`

Bulk-adds passwords from a newline-separated file — a `secretsdump.py` output, a Hashcat `--show` dump, a previous engagement's hit list.

```bash
kudzu import cracked_hashes.txt --tag secretsdump --rules
```

`--rules` and `--tag` apply to every password in the file.

### `kudzu where`

Prints the config and storage paths kudzu is currently using.

## Configuration — `~/.kudzu/config.yaml`

Created automatically on first run with these defaults:

```yaml
rules:
  year_range: true       # a 4-digit year (1800..current year) found in the
                           # password is expanded to every year from itself
                           # up to current_year + year.future_offset
  leet_speak: true        # full-string substitution: a->4, e->3, s->$...
  case_powerset: true     # every upper/lower combination of each letter
  symbol_wrap: true       # prepend / append a symbol
  digit_range: true       # a non-year digit run is replaced by every value
                           # within +/- digits_range.spread of it, same
                           # zero-padded width
  keyword_combo: true     # combine with --keyword, using the same year logic

year:
  future_offset: 2         # Shinra2022 -> up to Shinra<current_year + 2>
  short_form: true          # also emit the 2-digit year (Shinra26)
  separators: ["_", "-", "."]  # inserted right before the year (Shinra_2026)

digits_range:
  spread: 10   # Ete202 (not a year) -> Ete192 .. Ete212

case:
  max_length: 15   # skip full case-powerset expansion past this many
                     # letters (2^16+ combinations gets slow); the password
                     # is still included with its original casing

symbols: ["!", "@", "#", "$", ".", "?"]

leet:
  mapping: {a: "4", e: "3", i: "1", o: "0", s: "$", t: "7"}

keyword: null   # default --keyword value if not passed on the CLI

limits:
  max_variants_per_password: 5000   # output cap per stored password

storage:
  encryption: none   # "none" or "gpg"
```

Edit the file directly, or disable a rule you don't want (e.g. turn off `symbol_wrap` if the target policy doesn't require symbols).

The year range, digit range, and keyword-combo results are always kept in full regardless of the cap — they're the direct, bounded output of a rule you asked for. Only the case-powerset / leet / symbol-wrap combinatorial expansion on top of them is trimmed against `limits.max_variants_per_password` if it grows past it.

### Storage file format — `~/.kudzu/password/list.txt`

One entry per line, tab-separated:

```
<password>\t<tag>\t<ISO-8601 timestamp>\t<apply_rules: 0|1>
```

`tag` and `timestamp` may be empty. The file is plaintext by default — see [Advanced Configuration](#advanced-configuration) below to enable GPG encryption.

## Mutation Rules Reference

| Rule | Example input | Example outputs |
|---|---|---|
| Year range (1800..current year, up to +2) | `Shinra2022` (run in 2026) | `Shinra2023` ... `Shinra2028`, `Shinra26`, `Shinra_2026`, `Shinra-26` |
| Digit range (non-year numbers, ±10, same width) | `Ete202` | `Ete192` ... `Ete212` |
| Leet speak | `password` | `p4$$w0rd` (full-string substitution) |
| Case powerset (every letter, both cases) | `Ete` | `ete`, `Ete`, `eTe`, `etE`, `EtE`, `ETe`, `eTE`, `ETE` |
| Symbol wrap (prefix and suffix) | `Ete202` | `!Ete202`, `Ete202!`, `@Ete202`, `Ete202@`... |
| Keyword combo (same year logic) | `Shinra2022` + `--keyword ACME` | `Acme2022` ... `Acme2028` |

All rules are toggled individually in `config.yaml`. Whether a rule runs at all on a given password is decided when it's added, via `--rules`.

## Pentest Use Cases

### Full audit scenario

```bash
# 1. Kerberoast a service account, crack the hash offline
GetUserSPNs.py SHINRA.LOCAL/juser -request -outputfile roast.txt
hashcat -a 0 -m 13100 roast.txt rockyou.txt

# 2. Feed the cracked password into kudzu
kudzu add "Shinra2022" --tag kerberoast --rules

# 3. A mailbox dump turns up a random, password-manager-generated secret —
#    stored, but left unmutated
kudzu add "eqYQ5_RXZtUYPJ" --tag "mail ashleigh.lewis"

# 4. Grow the human-chosen passwords into a targeted candidate list
kudzu create-list spray.txt --keyword Shinra

# 5. Spray carefully against the target (see lockout note below)
netexec smb shinra.local -u users.txt -p spray.txt --continue-on-success
```

### Good practice: spraying over brute forcing

- Prefer **one password against many accounts** (spraying) over many passwords against one account — it's far less likely to trigger lockout policies.
- Check the target's lockout threshold before spraying (e.g. for AD: `net accounts /domain` or `Get-ADDefaultDomainPasswordPolicy`) and always leave headroom under the bad-attempt counter before each round.
- Keep candidate lists **short and targeted** — a 900-word kudzu output beats a 14-million-line rockyou.txt run against live accounts.
- Space spray attempts out (respect the lockout observation window) and monitor authentication logs where you have visibility.

## Legal & Ethical Notice

kudzu is built for **authorized security assessments only**, carried out under a signed engagement letter / rules of engagement with the target organization. Running password attacks against systems you do not have explicit written authorization to test is illegal in most jurisdictions. You are responsible for how you use this tool — use it only within the scope of an authorized audit.

## Lockout Safety Notes

- Always confirm the target's lockout policy before spraying (threshold, observation window, lockout duration) — for AD specifically, watch `badPwdCount`.
- A single bad spray round can lock out a whole set of accounts — test against a small, known-safe account first if possible.
- Prefer short, high-confidence candidate lists (few seed passwords, `--rules` only on the ones with a human-guessable pattern) over maximalist dumps.
- Coordinate spray timing with the client / blue team per the rules of engagement, especially outside of agreed testing windows.

## Advanced Configuration

### Encrypted storage

Set `storage.encryption: gpg` in `config.yaml`. kudzu will transparently decrypt the store (prompting for a passphrase) on read and re-encrypt (prompting again, with confirmation) on every write, using `gpg --symmetric --cipher-algo AES256`. No plaintext copy is left on disk. Requires the system `gpg` binary.

### Tags

Every stored password can carry a free-form `--tag` (e.g. `kerberoast`, `smb`, `mail victor.davis`) so `kudzu list` keeps a trace of where each credential came from, and `kudzu remove --tag` can clear a whole batch at once.

## FAQ

**Why not just use rockyou.txt?** rockyou.txt is a generic breach corpus; kudzu builds a *targeted* list from passwords this specific organization is actually using, which is both far shorter (safer against lockouts) and more likely to hit in a spray.

**Why would I add a password without `--rules`?** Because mutating a random, high-entropy credential (e.g. generated by a password manager) produces thousands of variants with zero chance of matching anything else — it just bloats the output. Store it, skip `--rules`, and it still rides along in the final wordlist unchanged.

**How do I avoid locking out accounts?** Keep seed lists small, apply `--rules` selectively, check the target's lockout policy first, and always prefer spraying one candidate across many accounts over brute-forcing one account.

**Can I use kudzu for protocols beyond AD?** Yes — `create-list` just produces a flat wordlist file, usable with any tool that accepts one (Hydra, Medusa, NetExec/CrackMapExec, custom scripts, web login forms, etc.).

**Does kudzu send data anywhere?** No. Everything is local; there's no network call in the tool itself.

## Roadmap

- Per-tag rule profiles (different mutation behavior for `smb` vs `kerberoast` entries)
- `age`-based encryption as an alternative to GPG
- Heuristic auto-suggestion of whether `--rules` is worth it for a given password

## Contribution

Issues and pull requests are welcome at [github.com/ned0x1/kudzu](https://github.com/ned0x1/kudzu). Please keep the standard library footprint minimal (`typer` + `pyyaml` only), run `ruff`/`black` before submitting, and include a short rationale for any new default rule.

## License

MIT — see [LICENSE](LICENSE).

## Acknowledgments

Inspired by the broader password-attack tradition — [KnowsMore](https://github.com/helviojunior/knowsmore), [Hashcat](https://hashcat.net/hashcat/) rule-based attacks, and the long-standing practice of turning one cracked password into a targeted spray list.
