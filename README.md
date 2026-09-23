# Orbinum Validator Node Setup

Adım adım Orbinum testnet validator kurulumu. Kaynak:
https://docs.orbinum.network/validators/running-a-validator

Tüm komutları sırasıyla, bir önceki adımın çıktısını kontrol ederek çalıştır.

---

## Ön koşullar

- [Validator requirements](https://docs.orbinum.network/validators/requirements)'ı karşılayan bir sunucu
- Docker + Compose plugin ([kurulum](https://docs.orbinum.network/nodes/installation))
- Registry token gerekmiyor, image public

---

## 1. `node-deploy`'i klonla ve yapılandır

```bash
git clone https://github.com/orbinum/node-deploy.git
cd node-deploy/testnet/validator
cp .env.example .env
```

### `.env` dosyasını açmak

Bir terminal editörüyle aç. En kolayı `nano`:
```bash
nano .env
```
- Kaydetmek için: `Ctrl+O` → `Enter` → çıkmak için: `Ctrl+X`
- `nano` yoksa: `sudo apt install nano -y`

`vim` tercih edersen:
```bash
vim .env
```
- `i` ile insert moduna geç, düzenle, `Esc` sonra `:wq` yaz Enter'a bas

Sadece görüntülemek (düzenlemeden) için:
```bash
cat .env
```

`.env` dosyasını aç ve şu değerleri düzenle:

| Değişken | Değer | Neden |
|---|---|---|
| `VALIDATOR_NAME` | tanınabilir bir isim | Telemetry'de görünür |
| `VALIDATOR_NODE_KEY` | `openssl rand -hex 32` çıktısı | Sabit libp2p kimliğin |
| `TELEMETRY_URL` | varsayılanı bırak | Seviye `1`, validator adresini dashboard'a yayınlar |
| `METRICS_BIND` | private IP'n, ya da `127.0.0.1` | Varsayılan kapalı gelir, dışa açık şey olmaz |
| `PUBLIC_ADDR` | **boş bırak** | Node public adresini kendi bulur |
| `RESERVED_NODES` | **boş bırak** | Eklemeli değil — kısmi liste node'u izole eder |
| `SYNC_MODE` | **boş bırak** (bootstrap durumu hariç) | Full sync güvenli varsayılan |

Node key üretmek için:
```bash
openssl rand -hex 32
```

Bootnode eklemene gerek yok — `testnet-spec.json` zaten hepsini içeriyor ve Compose dosyası zaten mount ediyor.

> **Yeni bir sunucuda acele mi ediyorsun?** `SYNC_MODE=--sync warp` her bloğu tekrar oynatmayı atlar ama **sadece tamamen yeni bir volume'da** kullanılabilir ve node, sync noktasından öncesine ait blok gövdelerini tutmaz.

Düzenlemeyi bitirdikten sonra kontrol için:
```bash
cat .env
```
Özellikle `PUBLIC_ADDR` ve `RESERVED_NODES`'un boş kaldığından emin ol.

---

## 2. Firewall'ı aç

```bash
sudo ufw allow 30333/tcp   # P2P — internetten erişilebilir olmalı
sudo ufw allow 22/tcp      # SSH
sudo ufw enable
```

`9944` (RPC) veya `9615` (metrics) için kural **ekleme** — monitoring sunucun private network'teyse ve gerçekten `9615`'e ihtiyacı varsa o zaman ekle.

---

## 3. Node'u başlat

```bash
docker compose pull
docker compose up -d
docker compose logs -f orbinum-validator
```

Compose dosyası `--validator`, `--no-mdns`, chain spec ve telemetry flag'lerini zaten geçiyor — komut satırına ekleyecek bir şey yok.

Logların ilk satırları donanım benchmark'ı (kapatma). Ardından node'un kendi peer identity'sini, sonra sync sırasında import mesajlarını göreceksin. Chain head'e ulaşıp idle olana kadar bekle, sonra devam et.

`.env`'i sonradan değiştirirsen restart değil **recreate** gerekir:
```bash
docker compose up -d --force-recreate orbinum-validator
```

---

## 4. Sync durumunu doğrula

```bash
docker exec orbinum-validator curl -s -H 'Content-Type: application/json' \
  -d '{"id":1,"jsonrpc":"2.0","method":"system_health"}' \
  http://localhost:9944
```

Beklenen çıktı:
```json
{ "jsonrpc": "2.0", "result": { "peers": 4, "isSyncing": false, "shouldHavePeers": true }, "id": 1 }
```

`isSyncing: false` ve `peers >= 2` görmelisin. `peers: 0` ise `30333/tcp`'nin gerçekten dışarıdan erişilebilir olduğunu ve `RESERVED_NODES`'un boş olduğunu kontrol et.

> `docker exec` kullanmamızın sebebi: konteyner içinde RPC portu her zaman `9944`'tür, host'ta `RPC_PORT`'u değiştirmiş olsan bile. Bu port asla internete açık olmamalı — binary her role'de `--rpc-methods Unsafe` zorluyor.

---

## 5. Session key'leri üret

Önce SS58 adresinin hex public key'ini bul:
```bash
docker exec orbinum-validator orbinum-node key inspect <SENIN_SS58_ADRESIN>
```
Çıktıdaki `Public key (hex)` satırını kopyala (`0x` ile başlayan 64 hex karakter).

Sonra o hex'i owner olarak geçerek key'leri üret ve proof al:
```bash
docker exec orbinum-validator curl -s -H 'Content-Type: application/json' \
  -d '{"id":1,"jsonrpc":"2.0","method":"author_rotateKeysWithOwner","params":["<HESAP_HEX_ADRESIN>"]}' \
  http://localhost:9944
```

Beklenen çıktı:
```json
{"jsonrpc":"2.0","id":1,"result":{"keys":"0x…","proof":"0x…"}}
```

- `keys` — 128 hex karakter (64 byte): Aura (sr25519) + GRANDPA (ed25519) pubkey'lerin, runtime'ın belirttiği sırayla
- `proof` — 256 hex karakter (128 byte): her key için hesabın üzerinden atılmış bir imza

Private key'ler node'un keystore'una (`/data/chains/orbinum_testnet/keystore`) yazılır, sunucudan hiç çıkmaz. Sadece `keys` ve `proof` zincire gidecek.

> **Bu komutu sadece bir kez, senkronize bir node'da çalıştır.** Tekrar çalıştırırsan yeni bir çift üretilir ve daha önce kaydettiğin `keys`/`proof` geçersiz olur. Eğer kaydettikten sonra tekrar rotate edersen, yeni çiftle `session.setKeys`'i tekrar göndermen gerekir yoksa node bir sonraki session'da block üretmeyi durdurur.
>
> Çıktıda `proof` alanı **yoksa** node hâlâ sync oluyor demektir — `system_health` `isSyncing: false` deyince tekrar dene.

**`keys` ve `proof`'u kopyala, bir sonraki adımda lazım.**

---

## 6. `session.setKeys` gönder (manuel — Polkadot.js Apps)

Bu, validator hesabınla imzalanan on-chain bir extrinsic — adım 5'te owner olarak verdiğin hesabın aynısı. Önce [faucet](https://docs.orbinum.network/getting-started/faucet)'ten fonla.

1. https://polkadot.js.org/apps/ adresini aç, `wss://rpc-1.testnet.orbinum.io`'ya bağlan
2. **Developer → Extrinsics**
3. Validator hesabını seç → `session` → `setKeys(keys, proof)`
4. Adım 5'teki `keys`'i `keys` alanına yapıştır
5. Adım 5'teki `proof`'u `proof` alanına yapıştır
6. Submit ve sign

> Senin hesabın Aura key'in **değil**. Orbinum'da `ValidatorId` doğrudan `AccountId`'dir. Bu ilişki sadece genesis validator'ları için geçerliydi (onların hesapları Aura key'lerinden türetilmişti).

Extrinsic reddedilirse (`Session.InvalidProof`), sebep şunlardan biridir:
- `proof` boş/`0x00`
- `setKeys`'i imzalayan hesap, adım 5'te owner olarak verdiğin hesap değil
- kopyaladıktan sonra tekrar rotate ettin, `keys`/`proof` artık eşleşmiyor

Her durumda çözüm aynı: adım 5'i doğru hesap hex'iyle tekrar yap, yeni `keys`/`proof` ile tekrar gönder.

**Doğrula:** **Developer → Chain state** → `session.nextKeys(hesabın)` sorgula. Gönderdiğin `keys` ile aynı sonucu dönmeli. Boş dönerse extrinsic finalize olmamıştır — sonraki adım başarısız olur, `addValidator` da `NoSessionKeys` ile reddedilir.

---

## 7. Keystore'un zincirle eşleştiğini doğrula

`session.nextKeys` zincirin key'lerini bildiğini kanıtlar ama node'un private key'lerin yarısını **elinde tuttuğunu** kanıtlamaz — `author_rotateKeysWithOwner`'ı başka bir makinede çalıştırdıysan ya da konteyneri volume'suz recreate ettiysen bu ikisi ayrışır.

```bash
docker exec orbinum-validator curl -s -H 'Content-Type: application/json' \
  -d '{"id":1,"jsonrpc":"2.0","method":"author_hasSessionKeys","params":["<ADIM_5TEKI_KEYS_HEXI>"]}' \
  http://localhost:9944
```

`keys`'i adım 5'ten aldığın gibi tek `0x` string olarak, Aura+GRANDPA birleşik, ayraçsız geç.

Beklenen çıktı:
```json
{ "jsonrpc": "2.0", "result": true, "id": 1 }
```

- `true` → bu node, zincire kayıtlı key'ler için imza atabilir, kurulum tamam.
- `false` → keystore'da eksik, node hiçbir zaman block üretmeyecek. **Bu durum log'larda, telemetry'de, hiçbir yerde görünmez** — node senkron, peer'lı, sağlıklı görünür ama sırası geldiğinde her seferinde slotu atlar. Adım 5'e dön, **bu node'da** tekrar rotate et, sonra adım 6'yı yeni `keys`/`proof` ile tekrar gönder.

---

## Sonraki adımlar

`session.nextKeys` senin key'lerini dönüyor ve `author_hasSessionKeys` `true` veriyorsa `addValidator`'ın gerektirdiği duruma geldin:

- [Apply to Join the Set](https://docs.orbinum.network/validators/apply)
- [Relay Setup and Rewards](https://docs.orbinum.network/validators/relay-setup)
