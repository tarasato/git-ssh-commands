# Sign a file with an SSH key (send a file to someone so they can prove it is from you)

Run every command yourself. Replace `example_ed25519` with your key file name and `user@example.com` with the name you want the receiver to see.

What a signature gives the receiver:

| Check | Without a signature | With a signature |
|---|---|---|
| The file came from you | cannot prove | proved (only your private key can make the signature) |
| The file was not changed on the way | cannot tell | any changed byte fails the check |
| You can deny sending it later | yes | no |
| Other people can read the file | yes | **still yes**. A signature is not encryption |

You need OpenSSH 8.0 or newer (`ssh -V`). Git Bash, Windows OpenSSH and Arch all have it.

---

# Part 1: Arch Linux

## Sender: sign

```bash
ssh-keygen -Y sign -f ~/.ssh/example_ed25519 -n file report.pdf
```

- `-Y sign`: make a signature
- `-f`: your **private** key (asks for the passphrase if the key has one)
- `-n file`: namespace. A label inside the signature saying what it is for, so a signature made for files cannot be reused as, for example, a git commit signature (`-n git`). Sender and receiver must use the same value.
- Result: `report.pdf.sig` next to `report.pdf`. Send **both** files.

If the key is already loaded in your agent, you can sign with the public key and the agent does the work (no passphrase prompt):

```bash
ssh-keygen -Y sign -f ~/.ssh/example_ed25519.pub -n file report.pdf
```

Sign several files at once:

```bash
ssh-keygen -Y sign -f ~/.ssh/example_ed25519 -n file report.pdf photo.png notes.txt
```

## Sender: give the receiver your public key (once)

```bash
cat ~/.ssh/example_ed25519.pub
```

```bash
ssh-keygen -lf ~/.ssh/example_ed25519.pub
```

Send the first output (the public key). Read the fingerprint (`SHA256:...`) from the second command to the receiver over a **different channel** (phone call, in person). If someone swaps the public key on the way, the fingerprints will not match.

## Receiver: trust the sender's key (once)

1. Save the public key you received as a file, e.g. `sender.pub`, and check its fingerprint against the one the sender read to you:

```bash
ssh-keygen -lf sender.pub
```

2. Only if the fingerprints match, add it to your trusted list. One line per person: the name, `namespaces="file"` (limits this key to file signatures), then the public key line:

```bash
echo "user@example.com namespaces=\"file\" $(cat sender.pub)" >> ~/.ssh/allowed_signers
```

The line looks like this:

```
user@example.com namespaces="file" ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... example_ed25519
```

## Receiver: verify

```bash
ssh-keygen -Y verify -f ~/.ssh/allowed_signers -I user@example.com -n file -s report.pdf.sig < report.pdf
```

- `-f`: your trusted list
- `-I`: who you expect the file to be from (must match the name in `allowed_signers`)
- `-n file`: same namespace the sender used
- `-s`: the signature file
- `< report.pdf`: the file itself is read from standard input

Good result:

```
Good "file" signature for user@example.com with ED25519 key SHA256:...
```

Bad result (file changed, wrong key, or wrong namespace):

```
Signature verification failed: incorrect signature
Could not verify signature.
```

Check the exit code in scripts: `0` = good, anything else = bad.

```bash
ssh-keygen -Y verify -f ~/.ssh/allowed_signers -I user@example.com -n file -s report.pdf.sig < report.pdf && echo OK || echo FAILED
```

## Receiver: who signed this? (unknown sender)

```bash
ssh-keygen -Y find-principals -f ~/.ssh/allowed_signers -s report.pdf.sig
```

Prints the name from `allowed_signers` whose key made the signature, or nothing.

## Check the signature without a trusted list

Proves only that the file matches *some* key, not *whose* key. Use for a quick integrity test, never as proof of the sender.

```bash
ssh-keygen -Y check-novalidate -n file -s report.pdf.sig < report.pdf
```

---

# Part 2: Windows (Git Bash + Windows OpenSSH)

The commands are the same as Part 1. Git Bash has its own `ssh-keygen`; the Windows one is at `C:\Program Files\OpenSSH\ssh-keygen.exe`. Both work for signing a file with a key **file**.

## Sender: sign

```bash
ssh-keygen -Y sign -f ~/.ssh/example_ed25519 -n file report.pdf
```

Using the key loaded in the **Windows** agent (the agent from `ssh-agent.md`, Part 2) needs the Windows `ssh-keygen.exe`, because Git Bash's `ssh-keygen` cannot talk to the Windows agent:

```bash
"/c/Program Files/OpenSSH/ssh-keygen.exe" -Y sign -f "$(cygpath -w ~/.ssh/example_ed25519.pub)" -n file report.pdf
```

## Sender: show the public key and its fingerprint

```bash
cat ~/.ssh/example_ed25519.pub
```

```bash
ssh-keygen -lf ~/.ssh/example_ed25519.pub
```

## Receiver: trust the key (once)

```bash
ssh-keygen -lf sender.pub
```

```bash
echo "user@example.com namespaces=\"file\" $(cat sender.pub)" >> ~/.ssh/allowed_signers
```

## Receiver: verify

```bash
ssh-keygen -Y verify -f ~/.ssh/allowed_signers -I user@example.com -n file -s report.pdf.sig < report.pdf
```

The same in **PowerShell** (PowerShell has no `<` input redirection, so pipe the file in as bytes):

```powershell
Get-Content -AsByteStream report.pdf | ssh-keygen -Y verify -f $HOME\.ssh\allowed_signers -I user@example.com -n file -s report.pdf.sig
```

(Windows PowerShell 5.1: use `-Encoding Byte` instead of `-AsByteStream`.)

---

# Notes

## Use a separate signing key (recommended)

A key that logs in to GitHub/servers and a key that signs files have different risks. A separate key can be shared publicly as "my signing key" and replaced without touching your logins.

```bash
ssh-keygen -t ed25519 -C "example-signing" -f ~/.ssh/example_signing_ed25519
```

## If your key is lost or stolen

Anyone with the private key can sign as you. Tell every receiver to delete your line from their `allowed_signers`, then send them a new public key and fingerprint. Signatures made with the old key no longer verify for them.

## Many files: sign one checksum list

```bash
sha256sum report.pdf photo.png notes.txt > SHA256SUMS
```

```bash
ssh-keygen -Y sign -f ~/.ssh/example_ed25519 -n file SHA256SUMS
```

Receiver verifies `SHA256SUMS` first, then the files against it:

```bash
ssh-keygen -Y verify -f ~/.ssh/allowed_signers -I user@example.com -n file -s SHA256SUMS.sig < SHA256SUMS
```

```bash
sha256sum -c SHA256SUMS
```

## Other ways to sign

- **GPG**: `gpg --detach-sign report.pdf` / `gpg --verify report.pdf.sig report.pdf`. Common for software releases.
- **PDF signature** (Adobe Acrobat or a certificate from a certificate authority): the signature is stored inside the PDF and shown by PDF readers. Use for contracts and official documents.
