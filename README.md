# FLOP Validator Preparation and Application Guide (Devnet)

**English** | [Türkçe](Turkce-Kurulum.md)

> [!WARNING]
> This guide does not make you a validator. It prepares your server and gets you ready to apply.
> On devnet the FLOP team admits validators by hand and does not take applications right away
> ([FLOP's note](https://x.com/flop_labs/status/2119121264388526205)). Do not rent a big server
> before you are admitted. The devnet gives no rewards. Anyone who tries to sell you an
> "approval" or a "whitelist" spot is a scammer.

These steps were tested on 8 November 2026 on a freshly installed Ubuntu 22.04 server.

---

## System Requirements

| Part | FLOP's recommendation |
| --- | --- |
| CPU | 8 cores |
| RAM | 32 GB (64 GB for testnet) |
| Disk | 4 TB NVMe (at least 2 TB for testnet) |
| Network | 1 Gbps |
| Operating system | Ubuntu 22.04 or 24.04 (amd64) |

No GPU is needed. Do the preparation on the server that will run your validator: the key FLOP
sends after admission only opens on the server whose details you sent.

---

## 1. Connect to the Server

```bash
ssh root@SERVER_IP
```

- Replace `SERVER_IP` with your server's IP address. If it uses a different SSH port:
  `ssh -p PORT root@SERVER_IP`
- On the first connection, answer the question with `yes`.
- On Windows, follow my WSL guide first: https://x.com/UfukDegen/status/1944066889346429338
- If you logged in as a normal user, type `sudo -i` first.

Example output:

![Connect to the server](assets/01-ssh.png)

---

## 2. Update the System and Install the Basic Packages

```bash
apt update && apt upgrade -y
```

```bash
apt install -y curl jq age xz-utils
```

- If a purple window appears, press Enter.
- If you see `*** System restart required ***` at your next login, you do not need to restart.

Example output:

![Update the system](assets/02-update.png)

---

## 3. Name the Server

```bash
hostnamectl set-hostname your-validator-name
hostname
```

- Replace `your-validator-name` with a name made of lowercase letters, digits and hyphens. This
  name goes to FLOP, so do not change it later.

Example output:

![Name the server](assets/03-hostname.png)

---

## 4. Install Docker

```bash
curl -fsSL https://get.docker.com | sh
```

```bash
docker --version && docker compose version
```

- If Docker is already installed on your server, skip the first command.

Example output:

![Install Docker](assets/04-docker.png)

---

## 5. Install Node.js 24 and pnpm

```bash
cd /tmp
SUMS=$(curl -fsSL https://nodejs.org/dist/latest-v24.x/SHASUMS256.txt)
NODE_TGZ=$(echo "$SUMS" | awk '/linux-x64\.tar\.xz$/{print $2}')
curl -fsSLO "https://nodejs.org/dist/latest-v24.x/$NODE_TGZ"
echo "$SUMS" | grep " $NODE_TGZ\$" | sha256sum -c -
tar -xJf "$NODE_TGZ" -C /usr/local --strip-components=1
node -v
corepack enable
```

- You should see an `OK` line and a version that starts with `v24`.

Example output:

![Install Node.js](assets/05-node.png)

---

## 6. Install Tailscale

```bash
. /etc/os-release
curl -fsSL "https://pkgs.tailscale.com/stable/ubuntu/$VERSION_CODENAME.noarmor.gpg" -o /usr/share/keyrings/tailscale-archive-keyring.gpg
curl -fsSL "https://pkgs.tailscale.com/stable/ubuntu/$VERSION_CODENAME.tailscale-keyring.list" -o /etc/apt/sources.list.d/tailscale.list
apt update && apt install -y tailscale
tailscale status
```

- The last line should say `Logged out.` Do not run `tailscale up`: the key FLOP sends after
  admission adds your server to FLOP's network.

Example output:

![Install Tailscale](assets/06-tailscale.png)

---

## 7. Install cosign

```bash
cd /tmp
curl -fsSLO https://github.com/sigstore/cosign/releases/download/v3.1.4/cosign-linux-amd64
curl -fsSL https://github.com/sigstore/cosign/releases/download/v3.1.4/cosign_checksums.txt | grep ' cosign-linux-amd64$' | sha256sum -c -
install -m 755 cosign-linux-amd64 /usr/local/bin/cosign
```

- You should see `cosign-linux-amd64: OK`.

Example output:

![Install cosign](assets/07-cosign.png)

---

## 8. Download and Check the Package

```bash
mkdir -p /opt/flop-validator && cd /opt/flop-validator
curl -fsSLO https://explorer.devnet.flop.network/validator-downloads/validator-cs-be91fe83-0.3.0-devnet.1.tar.gz
echo "bf98e281b7d3217fb52e9aeba75886d2165e66d32f0159dbc3f1f2e998e30ba7  validator-cs-be91fe83-0.3.0-devnet.1.tar.gz" | sha256sum -c -
tar -xzf validator-cs-be91fe83-0.3.0-devnet.1.tar.gz
cd validator-cs-be91fe83
sha256sum -c SHA256SUMS | grep -c ': OK$'
node preflight.mjs
```

- You should see `OK`, then `26`, then `verified` on the last line.

Example output:

![Download and check the package](assets/08-package.png)

---

## 9. Check the Signatures

```bash
WF=https://github.com/flop-labs/flop-core/.github/workflows/build-services.yml@refs/heads/main
for IMG in \
  ghcr.io/flop-labs/flop-core/node-devnet@sha256:84498fd73db2464eb13088ab34d369ddf7cddd22a45c810d140c9ff8ec16c91d \
  ghcr.io/flop-labs/flop-core/da-validator@sha256:c6709bb5bba04978a7800daccd87bb2bf916edd34d7dbf09f84aba5be9aca949; do
  if cosign verify "$IMG" --certificate-oidc-issuer https://token.actions.githubusercontent.com --certificate-identity "$WF" >/dev/null 2>&1; then echo "OK: ${IMG%%@*}"; else echo "FAILED: ${IMG%%@*}"; fi
done
```

- Both lines should say `OK`. If you see `FAILED`, stop.

Example output:

![Check the signatures](assets/09-signatures.png)

---

## 10. Install the Dependencies

```bash
cd /opt/flop-validator/validator-cs-be91fe83
export COREPACK_ENABLE_DOWNLOAD_PROMPT=0
pnpm --version
pnpm install --frozen-lockfile --ignore-scripts
```

- You should see `12.14.0` and `Done in ...`. If you see `Update available!`, do not update pnpm.

Example output:

![Install the dependencies](assets/10-pnpm.png)

---

## 11. Create the Validator Account

```bash
pnpm onboard prepare
ls -la account.json .env
```

- `Public SS58 address` is your validator address. It is not a secret.
- The command prints two details, but FLOP asks for three (step 13).
- `account.json` and `.env` hold your secret key. Never share them.

Example output:

![Create the validator account](assets/11-prepare.png)

---

## 12. Back Up the Account

Leave the server (`exit`) and run these commands **on your own computer**:

```bash
mkdir -p ~/flop-validator-backup && chmod 700 ~/flop-validator-backup
scp root@SERVER_IP:/opt/flop-validator/validator-cs-be91fe83/account.json ~/flop-validator-backup/
scp root@SERVER_IP:/opt/flop-validator/validator-cs-be91fe83/.env ~/flop-validator-backup/
```

- If your server uses a different SSH port, add `-P PORT` right after `scp`.
- Then connect to the server again: `ssh root@SERVER_IP`

Example output:

![Back up the account](assets/12-backup.png)

---

## 13. The Details for FLOP

```bash
hostname
jq -r .ss58Address /opt/flop-validator/validator-cs-be91fe83/account.json
cat /etc/ssh/ssh_host_ed25519_key.pub
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

- FLOP asks for three details, none of them secret: the hostname, the SS58 address and the line
  that starts with `ssh-ed25519`.
- FLOP may ask you to confirm the `SHA256:` fingerprint on a second channel.
- Never share `/etc/ssh/ssh_host_ed25519_key`, the file without `.pub`.

Example output:

![The details for FLOP](assets/13-details.png)

---

## 14. Apply

1. **Interest form:** fill in the form at https://flop.finance/apply/validator. Choose
   **Self-hosted** as the setup type and **Testnet** as the timing.
2. **GitHub #10:** if you have no contact on the FLOP team, post this text with your own details
   in [#10](https://github.com/flop-labs/network/issues/10):

```text
Devnet validator onboarding request. I have no operations contact yet.

Hostname: <hostname>
Public SS58 address: <5...>
SSH host public key (encryption recipient only): <ssh-ed25519 AAAA...>

Prepared with validator-cs-be91fe83 0.3.0-devnet.1: the archive SHA-256 matches the docs,
SHA256SUMS and node preflight.mjs pass, both Cosign signatures verify and pnpm onboard prepare
is done. Tailscale is installed and logged out (Tailscale SSH will stay off) and age is
installed. Nothing is started or registered.

Host: <cores> cores, <RAM> GB RAM, <disk> NVMe, <location>.
I can confirm the host key fingerprint on a second channel.
This comment contains only public values.
```

- Posting is not an admission, and FLOP promises no response time. Follow
  [@flop_labs](https://x.com/flop_labs) for updates.

---

## After Admission (Not Tested Yet)

After admission, FLOP sends you a single-use Tailscale key, the stake and a `da-peers.json` file.
According to [FLOP's docs](https://docs.flop.network/docs/run-validator):

1. Save the key in the package folder as `tailnet-key.age` and run the `tailscale up` command FLOP
   gives you. Then run `tailscale set --ssh=false`.
2. Put `da-peers.json` in the package folder.
3. When FLOP says its access checks are done:

   ```bash
   cd /opt/flop-validator/validator-cs-be91fe83
   pnpm onboard join
   ```

   When you see the `Ready:` line, tell FLOP "ready".
4. After FLOP tells you that you are active, check it: `pnpm onboard verify`

- If you run `join` before admission, you get `tailscale failed; no signing material was
  printed`. This is normal.
- If `join` stops halfway, do not run it again, do not delete any files, and contact FLOP.
- Never run `docker compose down -v`.

---

Prepared by [@UfukDegen](https://x.com/UfukDegen)
