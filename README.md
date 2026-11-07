# Gods of Arena — Deployment

[Gods of Arena](https://github.com/turanberker/gods-of-arena) uygulamasının Kubernetes manifestleri. Kustomize ile yazılmıştır (`kubectl apply -k` içinde gelir, ek araç gerekmez).

```
base/                      # ortamdan bağımsız temel (doğrudan uygulanmaz)
├── kustomization.yaml
├── namespace.yaml         # gods-of-arena
├── config.yaml            # goa-config (gizli olmayan ayarlar)
├── infra/                 # PostgreSQL, Redis, Kafka (tek kopyalı StatefulSet + kalıcı disk)
└── apps/                  # her uygulama kendi klasöründe
    ├── auth/ heroes/ crafting/ inventory/ market/   # Java, HPA 2..5, PDB
    ├── battle/                               # Go, HPA 2..5, PDB (Redis ile koordine)
    ├── web/ admin/                           # nginx, HPA 2..4, PDB
    └── gateway/                              # Gateway API: Gateway + HTTPRoute'lar
overlays/
└── prod/                  # üretim: Secret, alan adları, imaj etiketleri
    ├── kustomization.yaml
    ├── secrets.env.example  # → secrets.env (git'e girmez)
    └── patches/             # config.yaml (Origin, çerez), routes.yaml (alan adları), gateway-k3s.yaml
k3s/traefik-config.yaml    # k3s Traefik'te Gateway API'yi açar (sunucuya bir kez kopyalanır)
.github/workflows/validate.yml  # PR'da kustomize build + kubeconform
.github/workflows/cd.yml        # main'e push: sunucuda git pull + compose/.env (ENV_* değerlerinden)
```

## Küçük sunucu: docker compose

Küçük, tek bir sunucu için k3s yerine `compose/` klasöründeki docker compose kurulumu kullanılır (her servisten bir kopya, Caddy ile otomatik HTTPS). Rehber: [compose/README.md](compose/README.md).

## Hedef: tek VM üzerinde k3s

`overlays/prod` tek bir sunucuda çalışan [k3s](https://k3s.io) için ayarlıdır. k3s'in hazır getirdikleri kullanılır:

| İhtiyaç | k3s'te karşılığı |
|---|---|
| Gateway API denetleyicisi | Hazır Traefik (`k3s/traefik-config.yaml` ile Gateway desteği açılır), GatewayClass `traefik` |
| Dış IP / 80 portu | ServiceLB: Traefik sunucunun 80 (ve 443) portunu dinler |
| Kalıcı disk | `local-path` StorageClass (veri sunucunun diskinde, `/var/lib/rancher/k3s/storage`) |
| HPA ölçümleri | Hazır `metrics-server` |

`base/` küme bağımsız kalır (`gatewayClassName: eg`); k3s'e özel tek fark `overlays/prod/patches/gateway-k3s.yaml`'dadır. Traefik'te Gateway dinleyici portu entryPoint'in kapsayıcı içi portuyla eşleşmek zorunda olduğu için orada `8000` yazar; dışarıdan yine 80'den girilir.

**Sunucu boyutu:** Uygulamaların toplam istekleri (HPA en az 2 kopya ile) yaklaşık 2,8 vCPU ve 4,7 GiB bellektir; k3s'in kendisiyle birlikte en az **4 vCPU / 8 GB** önerilir. Daha küçük sunucuda HPA `minReplicas` değerleri overlay'de 1'e çekilebilir (kesintisiz güncelleme `maxSurge: 1` ile yine sürer).

### Sunucu kurulumu (bir kez)

> Bu adımlar henüz gerçek bir sunucuda denenmedi.

```bash
# 1. k3s
curl -sfL https://get.k3s.io | sh -
sudo kubectl get nodes

# 2. Traefik'te Gateway API'yi aç (k3s klasördeki dosyayı kendisi uygular)
sudo cp k3s/traefik-config.yaml /var/lib/rancher/k3s/server/manifests/traefik-config.yaml
sudo kubectl -n kube-system rollout status deploy/traefik

# 3. Kontrol: Gateway API CRD'leri ve GatewayClass
sudo kubectl get crd gateways.gateway.networking.k8s.io httproutes.gateway.networking.k8s.io
sudo kubectl get gatewayclass
```

CRD'ler yoksa Gateway API'nin standart CRD'leri [gateway-api sürümlerinden](https://github.com/kubernetes-sigs/gateway-api/releases) (`standard-install.yaml`) kurulur. Sunucuda 80 ve 443 portları güvenlik duvarında açık olmalıdır.

## İmajlar

İmajları gods-of-arena reposundaki **Imajlar** workflow'u `main`'e her merge'de Docker Hub'a gönderir:

| Uygulama | İmaj |
|---|---|
| auth | `berkerturan/goa-authentication` |
| heroes | `berkerturan/goa-heroes` |
| crafting | `berkerturan/goa-crafting` |
| inventory | `berkerturan/goa-inventory` |
| market | `berkerturan/goa-market` |
| battle | `berkerturan/goa-battle` |
| web | `berkerturan/goa-web-client` |
| admin | `berkerturan/goa-admin-client` |

Etiketler: `:latest` (main'in son derlemesi) ve `:sha-<kısa-commit>`. Prod'un kullandığı etiket `overlays/prod/kustomization.yaml` içindeki `images:` bölümündedir. Geri alınabilir deploy için sabit etiket önerilir:

```bash
cd overlays/prod
kustomize edit set image berkerturan/goa-heroes=berkerturan/goa-heroes:sha-1f67886
```

Docker Hub repoları özelse kümeye bir `imagePullSecret` eklenmesi gerekir (şu an tanımlı değil).

**Frontend imajları:** `VITE_*` adresleri derleme anında gömülür. Bu manifestlerle çalışmaları için gods-of-arena reposunda Actions değişkenleri tanımlanmalıdır (`/gw` yönlendirmesi, aşağıda):

- `WEB_BUILD_ARGS`:
  ```
  VITE_API_BASE_URL=/gw/api
  VITE_HEROES_API_URL=/gw
  VITE_CRAFTING_API_URL=/gw
  VITE_INVENTORY_API_URL=/gw
  VITE_MARKET_API_URL=/gw
  VITE_BATTLE_API_URL=/gw
  VITE_BATTLE_WS_URL=ws://goa.example.com/gw
  ```
- `ADMIN_BUILD_ARGS`:
  ```
  VITE_ADMIN_AUTH_API_URL=/gw/api
  VITE_ADMIN_CRAFTING_API_URL=/gw
  VITE_ADMIN_HEROES_API_URL=/gw
  VITE_ADMIN_BATTLE_API_URL=/gw
  ```

## Kurulum

```bash
cp overlays/prod/secrets.env.example overlays/prod/secrets.env   # gerçek değerleri yazın
kubectl apply -k overlays/prod
kubectl -n gods-of-arena get pods
```

Uygulamadan önce kontrol için: `kubectl kustomize overlays/prod | less`.

## Alan adı ve HTTPS

`goa.example.com` yer tutucudur. Gerçek alan adı belli olunca şu üç yer birlikte değişir:

1. `overlays/prod/patches/routes.yaml`: HTTPRoute `hostnames`.
2. `overlays/prod/patches/config.yaml`: `ALLOWED_ORIGINS`.
3. gods-of-arena reposunda `WEB_BUILD_ARGS` içindeki `VITE_BATTLE_WS_URL`.

TLS henüz yok (Gateway yalnızca HTTP 80 dinliyor). Önerilen yol cert-manager + Let's Encrypt; ayrı bir iş olarak eklenecek. TLS gelince: Gateway'e HTTPS dinleyicisi (Traefik'te `websecure` entryPoint'i, kapsayıcı içi port 8443), `ALLOWED_ORIGINS` için `https://`, `GOA_COOKIE_SECURE: "true"` ve `VITE_BATTLE_WS_URL` için `wss://`.

## Secret yönetimi

`goa-secrets`, `overlays/prod/secrets.env` dosyasından üretilir; bu dosya `.gitignore`'dadır ve repoya girmez. Dosya, `kubectl apply` çalıştırılan yerde (sunucu veya CI) bulunmalıdır. Kalıcı çözüm seçenekleri:

- **Sunucuda dosya:** En basiti. `secrets.env` sunucuda bir kez oluşturulur, deploy betiği onu kullanır.
- **GitHub Actions secret'ları:** Deploy işi `secrets.env`'i secret'lardan üretip uygular.
- **Sealed Secrets / SOPS:** Şifreli hâli repoya girer; Argo CD gibi GitOps kurulumunda gerekir.

## Yönlendirme

| Adres | Hedef |
|---|---|
| `goa.<alan>/` | web |
| `admin.goa.<alan>/` | admin |
| `*/gw/api/**` | auth (`/api/**`) |
| `*/gw/heroes/**` | heroes (`/heroes/**`) |
| `*/gw/crafting/**` | crafting |
| `*/gw/inventory/**` | inventory |
| `*/gw/market/**` | market |
| `*/gw/battle/**` | battle (REST + WebSocket) |

`/gw` öneki, `/heroes` ve `/inventory` gibi yolların aynı zamanda arayüz sayfası olmasındandır. Arayüz ve servisler aynı kökenden açıldığı için CORS gerekmez. `/internal/**` uçları dışarı açılmaz; servisler birbirine küme içi adlarla gider (`http://heroes:9002`, `http://inventory:9004`).

## Çok kopya

- **Java servisleri ve frontend'ler** durumsuzdur: oturum Redis'te, veri PostgreSQL'de. HPA en az 2 kopya tutar, CPU %70 üstünde artırır. PDB bakım sırasında en az bir kopyayı ayakta tutar, `topologySpreadConstraints` kopyaları farklı düğümlere dağıtır. Aynı anda açılan kopyalarda Liquibase kilidi migration'ı tek kopyaya yaptırır.
- **Go savaş sunucusu da çok kopyalıdır** (HPA en az 2, PDB, rolling update). Eşleşme kuyruğu, maç sahipliği (Redis kirası) ve kopyalar arası WebSocket aktarımı (Redis pub/sub) Redis'tedir; yapışkan oturum gerekmez. Kapanan veya çöken pod'un maçları diğer kopya tarafından hamle kaydından devralınır, istemci yeniden bağlanır. Redis artık battle için zorunludur (açılışta erişilemezse pod başlamaz). Ayrıntı: gods-of-arena reposunda `docs/backend-services/battle.md#çoklu-instance`.
- **Kafka** (tek broker, KRaft) maç sonucu olaylarını taşır: Go battle outbox → `battle.match-finished.v1` → heroes (XP, yaralanma) ve inventory (arena ücret havuzu). Topic ilk yayında otomatik açılır. Kafka kapalıyken maçlar oynanır; olaylar battle'ın outbox tablosunda bekler ve Kafka gelince gönderilir.
- **PostgreSQL ve Redis** tek kopyalı StatefulSet'tir. Üretimde yönetilen servis veya operatör (CloudNativePG, Redis Sentinel) önerilir; o durumda `base/infra` yerine `goa-config`'teki adresleri değiştirmek yeterlidir.

## Otomatik deploy (CD)

`.github/workflows/cd.yml`: `main`'e her push'ta (veya Actions → **CD** → *Run workflow* ile elle) sunucuya SSH ile bağlanır, bu reponun sunucudaki kopyasında `git pull --ff-only` yapar ve `compose/.env` dosyasını GitHub'daki `ENV_` önekli değerlerden yeniden yazar (izin 600). Başka bir şey yapmaz: imaj çekme ve container yeniden başlatma yoktur (imajlar güncellenince gods-of-arena'daki Imajlar workflow'unun deploy işi bunu yapar).

Tanımlar repo ayarlarında yapılır: Settings → Secrets and variables → Actions.

**Bağlantı**

| Ad | Tür | Açıklama |
|---|---|---|
| `SSH_HOST` | Secret | Sunucu adresi (IP veya alan adı) |
| `SSH_USER` | Secret | SSH kullanıcısı (ör. `root`) |
| `SSH_KEY` | Secret | Özel anahtarın tamamı (`-----BEGIN` … `END-----` satırları dahil); açık anahtarı sunucuda `~/.ssh/authorized_keys`'te olmalı |
| `SSH_PORT` | Secret, isteğe bağlı | Boşsa `22` |
| `DEPLOY_PATH` | Variable, isteğe bağlı | Reponun sunucudaki klasörü; boşsa `/root/gods-of-arena-deployment` |

**`.env` değerleri:** adı `ENV_` ile başlayan her Secret ve Variable, öneki atılarak `.env`'e yazılır (`ENV_POSTGRES_PASSWORD` → `POSTGRES_PASSWORD`). Parolalar ve anahtarlar **Secret** (değeri sonradan görünmez, loglarda maskelenir), gizli olmayan ayarlar **Variable** (görülüp düzenlenebilir) olarak tanımlanır. Aynı ad ikisinde de varsa Secret kazanır. Yeni bir anahtar için workflow'a dokunmak gerekmez; yalnızca yeni bir `ENV_...` tanımlanır. Bir anahtarı `.env`'den çıkarmak için tanımı silinir.

| Secret | Variable |
|---|---|
| `ENV_POSTGRES_PASSWORD`, `ENV_REDIS_PASSWORD`, `ENV_INTERNAL_API_TOKEN`, `ENV_GOA_ADMIN_USERNAME`, `ENV_GOA_ADMIN_EMAIL`, `ENV_GOA_ADMIN_PASSWORD`, `ENV_GEMINI_API_KEY` | `ENV_GOA_DOMAIN`, `ENV_GOA_ADMIN_DOMAIN`, `ENV_ACME_EMAIL`, `ENV_IMAGE_TAG`, `ENV_ARENA_ENTRY_FEE`, `ENV_ARENA_DRAW_REFUND_PERCENT`, `ENV_JAVA_MEM_LIMIT`, `ENV_HTTP_PORT`, `ENV_HTTPS_PORT` |

Tam liste ve anlamları: [compose/.env.example](compose/.env.example). Sunucuda repo bir kez `git clone` ile o klasöre kurulmuş olmalıdır. Yeni değerlerin container'lara geçmesi için `docker compose up -d` gerekir (ana repodaki deploy işi bunu yapar).

### İleride: k3s

Hedef tek VM + k3s olunca iki yol var:

1. **CI'dan SSH ile apply** (Tennis League'deki `appleboy/ssh-action` yaklaşımı): gods-of-arena'daki Imajlar workflow'una `needs: build-push` ile bir deploy işi eklenir. İş sunucuya SSH ile bağlanır, bu repoyu çeker, imaj etiketini yeni `sha-<commit>` yapar ve `kubectl apply -k overlays/prod` çalıştırır. Gerekenler: `SSH_HOST`, `SSH_USER`, `SSH_KEY` secret'ları ve sunucuda `kubectl` erişimi (k3s'te `/etc/rancher/k3s/k3s.yaml`). Tek VM için en az parçalı yol budur.
2. **Argo CD (GitOps):** Kümeye Argo CD kurulur, bu repoyu izler. CI yeni etiketi bu repoya commit'ler (gods-of-arena'dan bu repoya yazma yetkili bir token gerekir), Argo CD kümeyi eşitler. Geçmiş ve geri alma Git'te tutulur, ama kümede ek bir bileşen ve Secret'lar için Sealed Secrets/SOPS gerekir.

Önerim: önce 1 ile başlamak, ortam büyüyünce 2'ye geçmek.
