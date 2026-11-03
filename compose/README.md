# Tek sunucu kurulumu (docker compose)

Küçük, tek bir sunucu için. Her servisten bir kopya çalışır; PostgreSQL ve Redis aynı makinededir. Caddy 80/443 portlarını dinler, Let's Encrypt sertifikasını kendisi alıp yeniler ve istekleri servislere dağıtır. k3s kurulumu (`overlays/prod`) daha büyük makine içindir; ikisi aynı Docker Hub imajlarını kullanır.

> Bu adımlar henüz gerçek sunucuda denenmedi. Caddy yönlendirmesi sahte servislerle, compose dosyası `docker compose config` ile doğrulandı.

```
compose/
├── docker-compose.yml   # caddy, postgres, redis, 5 Java servisi, battle, web, admin
├── Caddyfile            # HTTPS + /gw yönlendirmesi
└── .env.example         # → .env (alan adları, parolalar; git'e girmez)
```

## Adresler

| Adres | Ne |
|---|---|
| `https://goa.amateurtennisleague.com` | Oyun (web) |
| `https://admin.goa.amateurtennisleague.com` | Admin paneli |
| `…/gw/api/**` | auth (`/api/**`) |
| `…/gw/heroes/**`, `/gw/crafting/**`, `/gw/inventory/**`, `/gw/market/**` | ilgili Java servisi |
| `…/gw/battle/**` | battle (REST + WebSocket) |

`/gw` öneki kaldırılarak iletilir (`/gw/heroes/x` → `heroes:9002/heroes/x`). Arayüz ve API aynı adreste; yine de Java servisleri `GOA_SECURITY_CORS_ALLOWED_ORIGINS` ile bu iki alan adına (ve mobil kabuğa) açıkça izin verir, yoksa POST istekleri 403 "Invalid CORS request" alabilir. `/internal/**` uçları dışarı açılmaz. Kök alan adı (`amateurtennisleague.com`, `www`) bu kurulumda kullanılmaz.

## Sunucu boyutu

Bellek sınırları: 5 Java servisi × 512 MB, battle 256 MB, Redis 128 MB, PostgreSQL, Caddy ve iki nginx ile toplam en fazla yaklaşık **3–3,5 GB**. Boşta gerçek kullanım daha düşüktür.

| Makine | Durum |
|---|---|
| 4 GB+ RAM, 2 vCPU | Rahat çalışır. |
| 2 GB RAM | `.env`'de `JAVA_MEM_LIMIT=384m` yapın ve en az 2 GB swap açın (aşağıda). İlk açılış yavaş olur. |
| 1 GB RAM | Yetmez. |

İlk açılışta Java servisleri Liquibase migration'larını çalıştırır; küçük makinede birkaç dakika sürebilir.

## 1. DNS kayıtları

Alan adı panelinde (amateurtennisleague.com'u nereden aldıysan) sunucunun public IP'sine iki kayıt:

| Tür | Ad | Değer |
|---|---|---|
| A | `goa` | sunucu IPv4 |
| A | `admin.goa` | sunucu IPv4 |

Sunucunun IPv6'sı varsa aynı adlarla `AAAA` kayıtları da eklenebilir (yanlış IPv6 sertifika almayı bozar; emin değilsen ekleme). Kontrol:

```bash
dig +short goa.amateurtennisleague.com
dig +short admin.goa.amateurtennisleague.com
```

Sunucu evdeyse (sabit IP yok, modem arkasında): modemde 80 ve 443'ü sunucuya yönlendir (port forwarding); IP değişiyorsa alan adı sağlayıcısının dinamik DNS'ini kullan.

## 2. Sunucu hazırlığı (bir kez)

Ubuntu/Debian için:

```bash
# Docker + compose eklentisi
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER   # sonra oturumu kapatıp aç

# Güvenlik duvarı: SSH, HTTP, HTTPS
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 443/udp
sudo ufw enable

# (2 GB RAM'li makinede) 2 GB swap
sudo fallocate -l 2G /swapfile && sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Bulut sağlayıcısında (Hetzner, DigitalOcean vb.) ayrıca bir güvenlik duvarı varsa orada da 80/443 açılmalıdır.

## 3. Frontend imajlarını bu alan adına göre derlet

Frontend'ler API adreslerini derleme anında gömer. gods-of-arena reposunda **Settings > Secrets and variables > Actions > Variables** altında iki değişken tanımla:

`WEB_BUILD_ARGS`:
```
VITE_API_BASE_URL=/gw/api
VITE_HEROES_API_URL=/gw
VITE_CRAFTING_API_URL=/gw
VITE_INVENTORY_API_URL=/gw
VITE_MARKET_API_URL=/gw
VITE_BATTLE_API_URL=/gw
VITE_BATTLE_WS_URL=wss://goa.amateurtennisleague.com/gw
```

`ADMIN_BUILD_ARGS`:
```
VITE_ADMIN_AUTH_API_URL=/gw/api
VITE_ADMIN_CRAFTING_API_URL=/gw
VITE_ADMIN_HEROES_API_URL=/gw
VITE_ADMIN_BATTLE_API_URL=/gw
```

Sonra **Actions > Imajlar > Run workflow** ile imajları yeniden derlet. Bu yapılmazsa arayüzler `localhost:900x`'e istek atar ve çalışmaz.

## 4. Kurulum

```bash
git clone https://github.com/turanberker/gods-of-arena-deployment.git
cd gods-of-arena-deployment/compose

cp .env.example .env && chmod 600 .env
nano .env    # ACME_EMAIL, POSTGRES_PASSWORD, REDIS_PASSWORD, INTERNAL_API_TOKEN, ilk admin
# Parola üretmek için: openssl rand -base64 32 | tr -d '/+='

docker compose pull
docker compose up -d
docker compose ps          # hepsi "healthy" olana kadar bekle
docker compose logs -f caddy   # "certificate obtained successfully" görülmeli
```

Sonra `https://goa.amateurtennisleague.com` ve `https://admin.goa.amateurtennisleague.com` açılır. İlk admin `.env`'deki `GOA_ADMIN_*` ile yalnızca ilk açılışta eklenir.

Sertifika alınamıyorsa: DNS kaydı sunucuyu göstermiyor ya da 80/443 dışarıdan kapalıdır. Çok sayıda başarısız deneme Let's Encrypt'in saatlik sınırına takılır; önce DNS'i `dig` ile doğrula.

## Secret'lar

Tüm gizli değerler sunucudaki `compose/.env` dosyasındadır; repoya girmez (`.gitignore`). Zorunlu olanlar boşsa `docker compose` başlamadan hata verir.

| Değişken | Açıklama |
|---|---|
| `GOA_DOMAIN`, `GOA_ADMIN_DOMAIN` | Oyun ve admin adresleri |
| `ACME_EMAIL` | Let's Encrypt bildirim e-postası |
| `POSTGRES_PASSWORD`, `REDIS_PASSWORD` | Veritabanı ve Redis parolaları. Postgres parolası ilk açılışta veritabanına yazılır; sonradan `.env`'de değiştirmek tek başına yetmez. |
| `INTERNAL_API_TOKEN` | Servisler arası iç çağrı anahtarı |
| `GOA_ADMIN_USERNAME/EMAIL/PASSWORD` | İlk admin hesabı |
| `GEMINI_API_KEY` | İsteğe bağlı; boşsa yedek isim listesi |

`.env`'in bir kopyasını güvenli bir yerde (parola yöneticisi) sakla.

## Güncelleme ve geri alma

```bash
cd gods-of-arena-deployment && git pull
cd compose
docker compose pull && docker compose up -d
docker image prune -f
```

Sabit sürüm için `.env`'de `IMAGE_TAG=sha-<kısa-commit>` (Docker Hub'daki etiket); geri almak için eski etikete dönüp `docker compose up -d`.

## Yedek

```bash
# Veritabanı dökümü
docker compose exec -T postgres pg_dump -U goa gods_of_arena | gzip > goa-$(date +%F).sql.gz
# Geri yükleme (boş veritabanına)
gunzip -c goa-2026-11-02.sql.gz | docker compose exec -T postgres psql -U goa gods_of_arena
```

Günlük yedek için bu komut cron'a eklenebilir; yedeği sunucu dışına da kopyala.

## 80/443 başka bir uygulamadaysa

Sunucuda Tennis League gibi 80/443'ü kullanan başka bir proxy (nginx, Caddy, Traefik) varsa iki Caddy aynı porta bağlanamaz. Seçenekler:

1. **Önerilen:** Bu Caddy'yi ana proxy yap, diğer uygulamanın adresini de `Caddyfile`'a ekle (`amateurtennisleague.com { reverse_proxy <kapsayıcı>:<port> }`, iki compose aynı Docker ağına bağlanır).
2. Bu Caddy'yi iç porta al (`.env`'de `HTTP_PORT=8080`, `HTTPS_PORT=8443`) ve mevcut proxy'den `goa.*` ile `admin.goa.*` adreslerini `http://127.0.0.1:8080`'e yönlendir. Bu durumda sertifikayı mevcut proxy alır; Caddyfile'daki site adreslerinin başına `http://` yazılmalıdır.

## Mobil uygulama

Mobil APK workflow'u (gods-of-arena > Actions > Mobil APK) şu an yerel ağdaki bir IP'ye ve `900x` portlarına göre derliyor; bu kurulumdaki `https://goa.amateurtennisleague.com/gw` adresine göre derlemek ayrı bir iş. Savaş sunucusunun Origin listesinde Capacitor kökenleri (`https://localhost`, `capacitor://localhost`) şimdiden var.
