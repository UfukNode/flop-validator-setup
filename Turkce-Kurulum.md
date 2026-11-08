# FLOP Validatör Hazırlık ve Başvuru Rehberi (Devnet)

**Türkçe** | [English](README.md)

> [!WARNING]
> Bu rehber sizi validatör yapmaz, sunucunuzu hazırlayıp başvurmanızı sağlar. Devnet'te
> validatörleri FLOP ekibi elle onaylıyor ve başvuruları hemen almıyor
> ([FLOP'un açıklaması](https://x.com/flop_labs/status/2119121264388526205)). Onay gelmeden büyük
> bir sunucu kiralamayın. Devnet ödül vermez.

Adımlar 8 Kasım 2026'da sıfırdan kurulan bir Ubuntu 22.04 sunucuda denendi.

---

## Sistem Gereksinimleri

| Parça | Devnet ([FLOP belgesi](https://docs.flop.network/docs/run-validator)) | Testnet ([FLOP testnet sayfası](https://flop.finance/testnet/)) |
| --- | --- | --- |
| İşlemci | 8 ayrılmış (dedicated) çekirdek | 8+ çekirdek |
| RAM | 32 GB | 64 GB |
| Disk | 4 TB NVMe (DA verisi dahil) | 4 TB NVMe (en az 2 TB) |
| İnternet | Güvenilir, düşük gecikmeli 1 Gbps | 1 Gbps, yedekli |
| İşletim sistemi | Linux amd64 | Belirtilmemiş |

- FLOP devnet değerlerinin ölçülmüş alt sınır değil planlama tahmini olduğunu, testnet
  değerlerinin de testnet açılmadan değişebileceğini söylüyor. GPU gerekmez.
- Bu rehberdeki komutlar Ubuntu için yazıldı.
- Hazırlığı validatörü çalıştıracağınız sunucuda yapın: FLOP'un onaydan sonra göndereceği anahtar
  sadece bilgilerini gönderdiğiniz sunucuda açılır.

---

## 1. Sunucuya Bağlanma

```bash
ssh root@SUNUCU_IP
```

- `SUNUCU_IP` yerine sunucunuzun IP adresini yazın. Farklı bir SSH portu varsa:
  `ssh -p PORT root@SUNUCU_IP`
- İlk bağlantıda çıkan soruya `yes` yazın.
- Windows kullanıyorsanız önce WSL rehberimi takip edin:
  https://x.com/UfukDegen/status/1944066889346429338
- Normal bir kullanıcıyla bağlandıysanız önce `sudo -i` yazın.

Örnek çıktı:

![Sunucuya bağlanma](assets/01-ssh.png)

---

## 2. Sistemi Güncelleme ve Gerekli Paketler

```bash
apt update && apt upgrade -y
```

```bash
apt install -y curl jq age xz-utils
```

- Mor bir pencere çıkarsa Enter'a basın.
- Sonraki girişte `*** System restart required ***` yazısını görürseniz yeniden başlatmanız
  gerekmez.

Örnek çıktı:

![Sistemi güncelleme](assets/02-update.png)

---

## 3. Sunucuya İsim Verme

```bash
hostnamectl set-hostname validator-ismi-girin
hostname
```

- `validator-ismi-girin` yerine küçük harf, rakam ve tireden oluşan bir isim yazın. Bu isim
  FLOP'a gönderilir, sonra değiştirmeyin.

Örnek çıktı:

![Sunucuya isim verme](assets/03-hostname.png)

---

## 4. Docker Kurulumu

```bash
curl -fsSL https://get.docker.com | sh
```

```bash
docker --version && docker compose version
```

- Docker sunucunuzda zaten kuruluysa ilk komutu atlayın.

Örnek çıktı:

![Docker kurulumu](assets/04-docker.png)

---

## 5. Node.js 24 ve pnpm Kurulumu

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

- `OK` satırını ve `v24` ile başlayan bir sürüm görmelisiniz.

Örnek çıktı:

![Node.js kurulumu](assets/05-node.png)

---

## 6. Tailscale Kurulumu

```bash
. /etc/os-release
curl -fsSL "https://pkgs.tailscale.com/stable/ubuntu/$VERSION_CODENAME.noarmor.gpg" -o /usr/share/keyrings/tailscale-archive-keyring.gpg
curl -fsSL "https://pkgs.tailscale.com/stable/ubuntu/$VERSION_CODENAME.tailscale-keyring.list" -o /etc/apt/sources.list.d/tailscale.list
apt update && apt install -y tailscale
tailscale status
```

- Son satırda `Logged out.` yazmalı. `tailscale up` çalıştırmayın: sunucuyu FLOP'un ağına onaydan
  sonra FLOP'un göndereceği anahtar ekler.

Örnek çıktı:

![Tailscale kurulumu](assets/06-tailscale.png)

---

## 7. cosign Kurulumu

```bash
cd /tmp
curl -fsSLO https://github.com/sigstore/cosign/releases/download/v3.1.4/cosign-linux-amd64
curl -fsSL https://github.com/sigstore/cosign/releases/download/v3.1.4/cosign_checksums.txt | grep ' cosign-linux-amd64$' | sha256sum -c -
install -m 755 cosign-linux-amd64 /usr/local/bin/cosign
```

- `cosign-linux-amd64: OK` görmelisiniz.

Örnek çıktı:

![cosign kurulumu](assets/07-cosign.png)

---

## 8. Paketi İndirme ve Doğrulama

```bash
mkdir -p /opt/flop-validator && cd /opt/flop-validator
curl -fsSLO https://explorer.devnet.flop.network/validator-downloads/validator-cs-be91fe83-0.3.0-devnet.1.tar.gz
echo "bf98e281b7d3217fb52e9aeba75886d2165e66d32f0159dbc3f1f2e998e30ba7  validator-cs-be91fe83-0.3.0-devnet.1.tar.gz" | sha256sum -c -
tar -xzf validator-cs-be91fe83-0.3.0-devnet.1.tar.gz
cd validator-cs-be91fe83
sha256sum -c SHA256SUMS | grep -c ': OK$'
node preflight.mjs
```

- Sırasıyla `OK`, `26` ve son satırda `verified` görmelisiniz.

Örnek çıktı:

![Paketi indirme ve doğrulama](assets/08-package.png)

---

## 9. İmzaları Doğrulama

```bash
WF=https://github.com/flop-labs/flop-core/.github/workflows/build-services.yml@refs/heads/main
for IMG in \
  ghcr.io/flop-labs/flop-core/node-devnet@sha256:84498fd73db2464eb13088ab34d369ddf7cddd22a45c810d140c9ff8ec16c91d \
  ghcr.io/flop-labs/flop-core/da-validator@sha256:c6709bb5bba04978a7800daccd87bb2bf916edd34d7dbf09f84aba5be9aca949; do
  if cosign verify "$IMG" --certificate-oidc-issuer https://token.actions.githubusercontent.com --certificate-identity "$WF" >/dev/null 2>&1; then echo "OK: ${IMG%%@*}"; else echo "FAILED: ${IMG%%@*}"; fi
done
```

- İki satırda da `OK` görmelisiniz. `FAILED` görürseniz devam etmeyin.

Örnek çıktı:

![İmzaları doğrulama](assets/09-signatures.png)

---

## 10. Bağımlılıkları Kurma

```bash
cd /opt/flop-validator/validator-cs-be91fe83
export COREPACK_ENABLE_DOWNLOAD_PROMPT=0
pnpm --version
pnpm install --frozen-lockfile --ignore-scripts
```

- `12.14.0` ve `Done in ...` görmelisiniz. `Update available!` yazısı çıkarsa pnpm'i
  güncellemeyin.

Örnek çıktı:

![Bağımlılıkları kurma](assets/10-pnpm.png)

---

## 11. Validatör Hesabını Oluşturma

```bash
pnpm onboard prepare
ls -la account.json .env
```

- `Public SS58 address` validatör adresinizdir, gizli değildir.
- Komut iki bilgi yazar ama FLOP üç bilgi ister (13. adım).
- `account.json` ve `.env` gizli anahtarınızı içerir. Kimseyle paylaşmayın.

Örnek çıktı:

![Validatör hesabını oluşturma](assets/11-prepare.png)

---

## 12. Yedek Alma

Sunucudan çıkın (`exit`) ve şu komutları **kendi bilgisayarınızda** çalıştırın:

```bash
mkdir -p ~/flop-validator-backup && chmod 700 ~/flop-validator-backup
scp root@SUNUCU_IP:/opt/flop-validator/validator-cs-be91fe83/account.json ~/flop-validator-backup/
scp root@SUNUCU_IP:/opt/flop-validator/validator-cs-be91fe83/.env ~/flop-validator-backup/
```

- Farklı bir SSH portu varsa `scp` kelimesinden sonra `-P PORT` ekleyin.
- Sonra sunucuya tekrar bağlanın: `ssh root@SUNUCU_IP`

Örnek çıktı:

![Yedek alma](assets/12-backup.png)

---

## 13. FLOP'a Gönderilecek Bilgiler

```bash
hostname
jq -r .ss58Address /opt/flop-validator/validator-cs-be91fe83/account.json
cat /etc/ssh/ssh_host_ed25519_key.pub
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

- FLOP üç bilgi ister, üçü de gizli değildir: hostname, SS58 adresi ve `ssh-ed25519` ile
  başlayan satır.
- `SHA256:` ile başlayan parmak izini FLOP sizden ikinci bir kanaldan teyit etmenizi isteyebilir.
- Sonu `.pub` olmayan `/etc/ssh/ssh_host_ed25519_key` dosyasını asla paylaşmayın.

Örnek çıktı:

![FLOP'a gönderilecek bilgiler](assets/13-details.png)

---

## 14. Başvuru

1. **İlgi formu:** https://flop.finance/apply/validator adresindeki formu doldurun. Kurulum türü
   olarak **Self-hosted**, zaman olarak **Testnet** seçin.
2. **GitHub #10:** FLOP'un belgesine göre, FLOP ekibinden tanıdığınız yoksa
   [#10](https://github.com/flop-labs/network/issues/10) sayfasına yorum yazıp tanıştırılma
   istiyorsunuz. Şu metni kendi bilgilerinizle kullanabilirsiniz:

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

- Bu yorum onay anlamına gelmez. FLOP cevap süresi vermiyor ve şu ana kadar orada kimseye cevap
  vermedi. Güncellemeleri [@flop_labs](https://x.com/flop_labs) hesabından takip edin.

---

## Onaydan Sonra (Henüz Denenmedi)

FLOP onay verince size tek kullanımlık bir Tailscale anahtarı, stake ve `da-peers.json` dosyası
gönderir. [FLOP'un belgesine](https://docs.flop.network/docs/run-validator) göre:

1. Anahtarı paket klasörüne `tailnet-key.age` adıyla kaydedin ve FLOP'un verdiği `tailscale up`
   komutunu çalıştırın. Ardından `tailscale set --ssh=false` yazın.
2. `da-peers.json` dosyasını paket klasörüne koyun.
3. FLOP erişim kontrollerini bitirdiğini söyleyince:

   ```bash
   cd /opt/flop-validator/validator-cs-be91fe83
   pnpm onboard join
   ```

   `Ready:` satırını görünce FLOP'a "ready" yazın.
4. FLOP aktif olduğunuzu bildirince kontrol edin: `pnpm onboard verify`

- Onaydan önce `join` çalıştırırsanız `tailscale failed; no signing material was printed` hatası
  alırsınız, bu normaldir.
- `join` yarıda kalırsa tekrar çalıştırmayın, dosyaları silmeyin ve FLOP'a yazın.
- `docker compose down -v` komutunu asla çalıştırmayın.

---

Hazırlayan: [@UfukDegen](https://x.com/UfukDegen)
