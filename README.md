# ProffGymIA Backend (merkezi API + web paneli)

Akıllı spor salonu ekosisteminin merkezi API'si (Python FastAPI + SQLAlchemy).
Mobil uygulama (Flutter), masaüstü panel (PyQt) ve turnike bu API'ye bağlanır.

## Hızlı başlatma (önerilen)

- **Windows:** `baslat.bat` dosyasına çift tıkla.
- **macOS / Linux:** terminalde `./baslat.sh`

İlk açılışta kurulumu ve örnek veriyi kendisi yapar (birkaç dakika). Sonraki açılışlarda birkaç saniyede API'yi başlatır ve tarayıcıda **web panelini** açar (`http://127.0.0.1:8000/`). API dokümantasyonu `http://127.0.0.1:8000/docs` adresinde.

## Web paneli

`app/static/` klasöründe, derleme gerektirmeyen tek sayfalık panel. Giriş yapan kişinin rolüne göre değişir:

- **Yönetici:** Bugün (aktif üye, bitmek üzere olanlar, günlük giriş, aylık satış), Üyeler (arama, filtre, yeni kayıt), üye detayında paket satışı ve PT ataması, Paketler, Turnike (QR doğrulama, kamerayla okutma, geçiş kayıtları).
- **PT:** Danışanlarım, danışan detayında kilo grafiği, metabolizma, günlük makrolar, öğünler ve antrenman tanımlama.
- **Üye:** Kalan gün halkası ve 60 sn'de yenilenen QR kod, su takibi, metabolizma, kilo ve öğün girişi, antrenman programı, profil.

Giriş ekranındaki **Kayıt ol** her zaman üye hesabı açar ve hemen giriş yapar. Yönetici üyenin sayfasındaki
**Personal trainer yap** / **Üye yap** butonuyla rolü değiştirir. Hiç yönetici yoksa ilk hesap yönetici olur (*İlk kurulum*).
İlgili uçlar: `POST /auth/register`, `GET /auth/setup`, `POST /auth/setup-admin`, `PATCH /users/{id}/role`.

Renkler olimpik ağırlık plakalarından alındı ve her zaman anlam taşır: yeşil aktif/izin, sarı bitmek üzere, kırmızı süresi dolmuş/red, mavi işlem.

Turnikeyi denemek için: bir pencerede yönetici, gizli pencerede üye olarak giriş yap; üyenin Ana sayfasındaki **Kodu kopyala** ile alınan kodu Turnike ekranına yapıştır. Yazı tipleri, QR çizimi ve kamera okuma için internet bağlantısı gerekir.

## Elle kurulum

```bash
cd ProffGymIA/backend
python -m venv .venv
# Windows:  .venv\Scripts\activate      macOS/Linux:  source .venv/bin/activate
pip install -r requirements.txt     # ../../ortak klasöründeki ortak kodu da kurar

cp .env.example .env                # Windows: copy .env.example .env
python -m app.anahtar               # token imza anahtarını üretir (yoksa sunucu açılmaz)
python seed.py                      # örnek veri (isteğe bağlı)
uvicorn app.main:app --reload       # API'yi başlat
```

Tarayıcıda **http://127.0.0.1:8000/docs** adresini aç: tüm uçları buradan deneyebilirsin.
Sağ üstteki **Authorize** butonuna `admin@proffgymia.com` / `admin123` yazarak giriş yap.

Testleri çalıştırmak için: `pytest` (geçici bir veritabanı kullanır, gerçek verine dokunmaz)

### Örnek hesaplar (`seed.py` sonrası)

| Rol   | E-posta          | Şifre    | Not                                   |
|-------|------------------|----------|---------------------------------------|
| Admin | admin@proffgymia.com  | admin123 | Salon sahibi / resepsiyon             |
| PT    | pt@proffgymia.com     | pt1234   | Ayşe ve Can'ın eğitmeni               |
| Üye   | ayse@proffgymia.com   | uye1234  | Aktif üyelik, kilo veriyor            |
| Üye   | can@proffgymia.com    | uye1234  | Aktif üyelik, kilo alıyor             |
| Üye   | eski@proffgymia.com   | uye1234  | Üyelik bitmiş → turnike reddeder      |

## Klasör yapısı

```
app/
  main.py              Uygulama girişi, router'lar
  config.py            Ayarlar (.env ile değiştirilebilir)
  database.py          Veritabanı bağlantısı
  models.py            Tablolar
  schemas.py           İstek/yanıt şemaları
  security.py          bcrypt şifre hash'i, JWT ve QR token
  deps.py              Giriş kontrolü, rol yetkisi, PT→danışan erişim kuralı
  services/
    metabolism.py      Mifflin-St Jeor BMR, TDEE, makro, su, BMI (saf fonksiyonlar)
    membership.py      Kalan gün ve uzatma hesabı
    profile.py         Profil + son kilodan metabolizma üretimi
  routers/             auth, users, memberships, tracking, workouts, checkin
tests/                 Otomatik testler
seed.py                Örnek veri
```

## Tablolar

`users` · `packages` · `memberships` · `weight_logs` · `water_logs` · `meals` · `workouts` · `workout_exercises` · `check_ins`

`users.face_encoding` alanı 2. aşamadaki yüz tanıma için şimdiden ayrıldı.

## Yetki kuralları

- **Üye**: yalnızca kendi verisini görür ve girer.
- **PT**: kendi danışanlarının verisini `?user_id=` ile görür, onlara antrenman tanımlar.
- **Admin**: herkesi görür; üye/PT kaydeder, paket satar, PT atar.

## API uçları

| Metot | Yol | Kim | Ne yapar |
|---|---|---|---|
| POST | `/auth/register` | Herkes | Mobil üye kaydı |
| POST | `/auth/login` | Herkes | Token al (form: `username`=e-posta, `password`) |
| GET/PATCH | `/users/me` | Giriş yapan | Profil (cinsiyet, doğum tarihi, boy, aktivite, hedef) |
| GET | `/users/me/metabolism` | Giriş yapan | BMR, TDEE, hedef kalori, makro, su, BMI |
| GET/POST | `/users` | Personel / Admin | Üye listesi (arama, rol filtresi) / yeni kayıt |
| PATCH | `/users/{id}/trainer` | Admin | Üyeye PT ata |
| GET | `/packages` | Herkes | Paketler ve fiyatlar |
| POST/PATCH | `/packages` | Admin | Paket ekle/düzenle |
| GET | `/memberships/status` | Giriş yapan | Aktif mi, kalan gün |
| POST | `/memberships` | Admin | Paket sat / üyelik uzat |
| POST/GET | `/weights` | Üye / PT | Günlük tartı, kilo geçmişi (grafik için) |
| POST/GET | `/water` | Üye / PT | Su ekle, günlük toplam ve hedef |
| POST/GET | `/meals` | Üye / PT | Öğün ekle (kalori boş bırakılırsa yazılan öğünden hesaplanır), öğün listesi |
| POST | `/meals/photo` | Üye | Yemek fotoğrafı yükle |
| GET | `/meals/summary` | Üye / PT | Günlük makro toplamı ve hedefler |
| POST | `/ai/food-text` | Giriş yapan | Yazılan öğünden kalori ve makro, kaydetmez (`quick=true`: yazarken anlık önizleme) |
| POST/DELETE | `/users/me/photo` | Giriş yapan | Profil fotoğrafı yükle (400x400 kare kırpılır) / kaldır |
| POST/DELETE | `/users/{id}/photo` | Admin | Üyenin profil fotoğrafını yükle / kaldır |
| POST/GET | `/workouts` | PT / Üye | Antrenman tanımla / günün antrenmanı |
| GET | `/checkin/qr-token` | Üye | 60 sn geçerli turnike QR içeriği |
| POST | `/checkin/qr` | Personel / turnike | QR'ı doğrula → izin / red |
| GET | `/checkin/logs` | Admin | Geçiş kayıtları |
| POST | `/checkin/face` | Personel / turnike | Kameradan kare → yüz eşleştir → izin / red |
| POST/DELETE | `/users/{id}/face` | Admin | Üyenin yüzünü turnike için kaydet / sil |
| POST | `/meals/photo` | Üye | Fotoğraf yükle → yapay zeka makroları doldurur |
| POST | `/ai/food` | Giriş yapan | Fotoğrafı analiz et, kaydetme (önizleme) |
| GET | `/ai/status` | Giriş yapan | Etkin yapay zeka sağlayıcısı |
| POST/GET | `/form-sessions` | Üye / PT | Form analizi set sonucu (tekrar, doğru tekrar, uyarılar) |
| GET/POST/PATCH/DELETE | `/supplements` | Üye | Takviyeler, hatırlatma saatleri, stok |
| POST | `/supplements/{id}/take` | Üye | "Aldım": stoktan bir servis düşer |
| GET | `/reports/monthly` | Üye / PT / Admin | Aylık PDF gelişim karnesi (`?month=2026-10&user_id=`) |
| GET | `/overview/stats`, `/overview/members` | Personel | Panel özetleri |

## Hesaplama yöntemi

- **BMR (Mifflin-St Jeor)**: Erkek `10·kg + 6.25·cm − 5·yaş + 5`, Kadın `… − 161`
- **TDEE** = BMR × aktivite çarpanı (1.2 / 1.375 / 1.55 / 1.725 / 1.9)
- **Hedef kalori**: kilo ver −500, koru 0, kilo al +300 (kadın ≥1200, erkek ≥1500 alt sınır)
- **Protein**: kilo ver 2.0, koru 1.6, kilo al 1.8 g/kg · **Yağ**: kalorinin %25'i · **Karbonhidrat**: kalan
- **Su**: 35 ml/kg

## PostgreSQL'e geçiş

`pip install "psycopg[binary]"` ve proje klasörüne `.env` dosyası:

```
PROFFGYMIA_DATABASE_URL=postgresql+psycopg://kullanici:sifre@localhost:5432/proffgymia
PROFFGYMIA_CORS_ORIGINS=https://panel.salonum.com   # yalnızca ayrı adresten sunulan bir ön yüz varsa
```

`PROFFGYMIA_SECRET_KEY` satırını boş bırak ve `python -m app.anahtar` çalıştır; en az 32 karakterlik rastgele bir anahtar yazar.
Anahtar eksik veya zayıfsa sunucu açılmaz.

## Yapay zeka ayarları (`.env`)

İlk çalıştırmada `.env.example` dosyası `.env` olarak kopyalanır. Yemek analizi için içine bir anahtar yaz:

```
GEMINI_API_KEY=buraya-anahtar      # https://aistudio.google.com/apikey (ücretsiz)
```

Anahtar yoksa sistem **demo** modunda çalışır: fotoğraf kaydedilir, örnek bir tabak değeri döner (sunumda internet yoksa işe yarar).

Yüz tanıma modelleri (~37 MB) ilk yüz kaydında `models/` klasörüne otomatik iner.

## Veritabanı güncellemesi

Sunucu her açılışta eksik tabloları ve sütunları ekler (`database.ensure_schema`). 1. aşamada oluşan `proffgymia.db` silinmeden çalışır.
Örnek verileri sıfırdan yüklemek istersen sunucuyu kapat, `proffgymia.db` dosyasını sil ve `baslat.bat`'ı tekrar aç.
