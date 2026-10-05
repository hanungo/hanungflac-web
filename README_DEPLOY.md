# Deploy HanungFLAC — Azure Static Web Apps (+ custom domain)

Paket statis siap unggah. **Komputasi sepenuhnya di sisi klien (browser)** —
DEM & parameter tidak dikirim ke server. Pengecualian: saat tombol Interpretasi AI
ditekan, data hasil dikirim ke penyedia AI (Gemini / Copilot Studio). Hasil bersifat
**indikatif**; keputusan L3/L4 wajib diverifikasi & ditandatangani engineer geoteknik
berlisensi (Kepmen ESDM 1827/2018).

## Isi paket
| Berkas | Keterangan |
|---|---|
| `index.html` | Halaman utama = **versi terkunci** (AES-256). Password membuka alat FLAC-SSR penuh. |
| `geotiff/index.html` | Alat konverter GeoTIFF → ASC/DXF. |
| `staticwebapp.config.json` | Routing, header keamanan, `noindex`, gate Entra ID. |
| `robots.txt` | Larang indeks mesin pencari. |
| `.github/workflows/azure-static-web-apps.yml` | CI deploy (opsional; portal bisa auto-generate). |

> Versi **polos** (`geoflac_slope_tool.html`) dan **pengunci** (owner console) **sengaja
> tidak disertakan** agar tidak terekspos publik. Alat penuh tetap tersedia setelah
> password lock pada `index.html`.

## Dua lapis keamanan (bisa dipilih)
1. **Gate Entra ID** (platform) — `staticwebapp.config.json` mewajibkan pengguna login
   (`allowedRoles: ["authenticated"]`), pengunjung anonim diarahkan ke `/.auth/login/aad`.
2. **Password lock** (berkas) — AES-256 pada `index.html`.

- Mau **hanya password lock** (tanpa login): hapus blok `"routes"` dan `"responseOverrides"`
  dari `staticwebapp.config.json`.
- Mau **batasi hanya tenant Adaro**: lihat bagian "Pembatasan tenant" di bawah
  (butuh paket **Standard** + app registration di Entra Adaro).

---

## Langkah deploy (cara portal, paling cepat)
1. Unggah isi paket ini ke repo GitHub (mis. `hanungflac-web`), berkas di **root**.
2. Azure Portal → **Create a resource → Static Web App**.
   - Plan: **Free** (uji) atau **Standard** (produksi: custom auth, SLA, custom domain lebih fleksibel).
   - Source: **GitHub** → pilih repo & branch `main`.
   - Build presets: **Custom** → App location `/`, Api location kosong, Output location kosong.
3. Azure membuat workflow GitHub Actions & deploy otomatis. URL awal: `https://<nama>.azurestaticapps.net`.
4. Cek: buka URL → diarahkan login Entra → muncul halaman terkunci → masukkan password → alat jalan.
   Pastikan **Web Crypto** aktif (HTTPS otomatis) sehingga lock & solver berjalan.

### Alternatif: deploy via CLI (SWA CLI)
```bash
npm i -g @azure/static-web-apps-cli
swa deploy ./ --deployment-token <TOKEN_DARI_PORTAL> --env production
```

---

## Custom domain
1. SWA resource → **Custom domains → Add**.
2. **Subdomain** (disarankan, mis. `geoflac.adaro.co.id`):
   - Tambah **CNAME** → `<nama>.azurestaticapps.net` di DNS.
   - Validasi via **CNAME** atau **TXT** sesuai instruksi portal.
3. **Apex/root** (`contoh.id`): pakai **ALIAS/ANAME** jika registrar mendukung, atau
   delegasikan DNS ke **Azure DNS** lalu pakai alias record. (CNAME tak boleh di apex.)
4. TLS diterbitkan otomatis oleh Azure (gratis). Tunggu propagasi DNS.
5. Jika domain `*.adaro.*` → ajukan perubahan CNAME/TXT ke **IT Adaro**.

---

## Pembatasan tenant (hanya karyawan Adaro) — produksi
Entra bawaan (Free) mengizinkan akun Microsoft mana pun. Untuk membatasi ke tenant Adaro:
1. Paket **Standard**.
2. Daftarkan **App registration** di Entra ID Adaro (dapatkan Client ID; simpan Client Secret
   sebagai application setting `AAD_CLIENT_ID` / `AAD_CLIENT_SECRET`).
3. Tambahkan provider kustom di `staticwebapp.config.json`:
```json
"auth": {
  "identityProviders": {
    "azureActiveDirectory": {
      "registration": {
        "openIdIssuer": "https://login.microsoftonline.com/<TENANT_ID_ADARO>/v2.0",
        "clientIdSettingName": "AAD_CLIENT_ID",
        "clientSecretSettingName": "AAD_CLIENT_SECRET"
      }
    }
  }
}
```
Dengan issuer tenant Adaro, hanya akun tenant tsb. yang bisa login.

---

## Catatan CSP / AI
`staticwebapp.config.json` sengaja **tidak** memasang `Content-Security-Policy` ketat karena
alat memakai inline script, Blob URL (Web Worker), dan `fetch` ke API AI. Bila ingin CSP, uji
dulu dan izinkan minimal:
```
script-src 'self' 'unsafe-inline' blob:;
worker-src 'self' blob:;
connect-src 'self' https://generativelanguage.googleapis.com https://directline.botframework.com https://*.directline.botframework.com;
```

## Pembaruan versi
Ganti `index.html` (dan `geotiff/index.html`) dengan hasil build terbaru, commit ke `main` →
deploy otomatis. Catat nomor revisi sesuai **WIN-AI-GHL-02-010**. Owner: GHM – Quality & Geotechnical.
