# hr-promotion-api

Project hands-on: **Training → Flask API → Streamlit → Vercel**.
Model memprediksi apakah karyawan akan dipromosikan (dataset Human Capital, ~54.8k baris).

```
hr-promotion-api/
├── api/
│   ├── __init__.py
│   └── index.py            # Flask API: /health, /schema, /predict
├── models/                 # dihasilkan training.py (WAJIB di-commit untuk Vercel)
│   ├── model.pkl
│   ├── preprocessor.pkl
│   └── metadata.json       # metrik, threshold, daftar kategori, versi library
├── streamlit/
│   ├── app.py              # router halaman + sidebar (API URL)
│   ├── api_client.py       # semua request ke Flask API
│   ├── prediction_page.py  # halaman Prediction (form manual)
│   ├── profile_page.py     # halaman Employee Profiles (dari employees.csv)
│   ├── employees.csv       # 40 karyawan fiktif
│   └── requirements.txt
├── scripts/
│   ├── training.py         # latih + evaluasi + simpan model
│   └── collection.py       # uji API (lokal atau URL Vercel)
├── data/                   # cache dataset (di-ignore git)
├── requirements.txt        # dependensi API (dipakai Vercel)
├── vercel.json
├── .vercelignore
├── .gitignore
└── README.md
```

## Sesi 1 — Setup & training (Codespace)

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt -r streamlit/requirements.txt
python scripts/training.py
```

Diskusikan hasilnya: kelas promosi hanya ~8.5% sehingga **accuracy menyesatkan**.
Kita lihat ROC-AUC, F1, confusion matrix, dan memilih *threshold* di data validasi
(bukan di data test) agar evaluasi akhir jujur. Preprocessor di-`fit` hanya pada data train
(mencegah data leakage).

## Sesi 2 — Flask API

```bash
python api/index.py                 # terminal 1
python scripts/collection.py        # terminal 2 (atau tanpa server: tes lokal otomatis)
curl http://127.0.0.1:5000/health
curl -X POST http://127.0.0.1:5000/predict -H "Content-Type: application/json" -d '{
  "department":"Analytics","region":"region_2","education":"Master'"'"'s & above",
  "gender":"Male","recruitment_channel":"Sourcing","no_of_trainings":1,"age":30,
  "previous_year_rating":5,"length_of_service":4,"awards_won":1,"avg_training_score":92}'
```

Poin ajar: model dimuat sekali saat start, validasi input (400 vs 200),
`/schema` agar UI tidak menduplikasi daftar opsi, dan peringatan jika versi
scikit-learn saat serving berbeda dari saat training.

## Sesi 3 — Streamlit

```bash
streamlit run streamlit/app.py      # API harus sedang jalan, pastikan .venv aktif
```

Halaman: **Home**, **Prediction** (form manual), dan **Employee Profiles**
(pilih karyawan dari `streamlit/employees.csv` → profile card, assessment, lalu `POST /predict`;
ada juga tombol prediksi massal 40 karyawan). Semua halaman memanggil API lewat `api_client.py`.

Di Codespace, port 5000 dan 8501 akan muncul di tab **Ports**. Karena Streamlit dan API
berjalan di mesin yang sama, `http://127.0.0.1:5000` tetap bisa dipakai.

## Sesi 3b — Tunnel dengan ngrok (API di Codespace → Streamlit Cloud)

Berguna sebelum Vercel: API tetap jalan di Codespace, tapi bisa diakses dari internet.

### Install ngrok (Codespace / Linux)

```bash
uname -m    # x86_64 -> pakai amd64 | aarch64 -> ganti "amd64" di URL dengan "arm64"
curl -sSLo ngrok.tgz https://bin.equinox.io/c/bNyj1mQVY4c/ngrok-v3-stable-linux-amd64.tgz
sudo tar -xzf ngrok.tgz -C /usr/local/bin
rm ngrok.tgz
ngrok version    # pastikan versi muncul
```

### Jalankan tunnel

```bash
# 1. Daftarkan authtoken akun gratis (https://dashboard.ngrok.com)
ngrok config add-authtoken <TOKEN_ANDA>

# 2. Jalankan API (terminal 1) lalu tunnel (terminal 2)
python api/index.py
ngrok http 5000
```

ngrok menampilkan URL seperti `https://xxxx.ngrok-free.app`. Pakai URL itu:
- Uji: `python scripts/collection.py --url https://xxxx.ngrok-free.app`
- Streamlit lokal: tempel ke kolom **API URL** di sidebar, atau `API_URL=https://xxxx.ngrok-free.app streamlit run streamlit/app.py`
- Streamlit Cloud: isi secret `API_URL` dengan URL tersebut.

Catatan: URL ngrok gratis **berubah setiap tunnel dibuat ulang**, dan mati jika Codespace/terminal berhenti.
Header `ngrok-skip-browser-warning` sudah ditambahkan di `app.py` dan `collection.py`
agar tidak terblokir halaman peringatan ngrok.

## Sesi 4 — Deploy

**API → Vercel**
1. Push project ke GitHub (pastikan `models/*.pkl` ikut ter-commit).
2. vercel.com → *Add New Project* → pilih repo (Framework: Other). Atau CLI: `npm i -g vercel && vercel --prod`.
3. Uji: `python scripts/collection.py --url https://<project>.vercel.app`

**Streamlit → Streamlit Community Cloud**
1. share.streamlit.io → pilih repo, *Main file path*: `streamlit/app.py`.
2. Di *Advanced settings → Secrets*: `API_URL = "https://<project>.vercel.app"`
3. Streamlit Cloud membaca `requirements.txt` di folder yang sama dengan app (`streamlit/requirements.txt`).

## Catatan penting (gotcha)

- **Versi library harus sama** antara training dan serving. Itu alasan model lama
  (`sklearn==1.2.2`, Python 3.10) tidak bisa dipakai di Vercel: kita latih ulang.
  Setelah mengganti versi di `requirements.txt`, jalankan ulang `training.py` dan commit `models/`.
- Tier warna gauge (>80% hijau, 55–80% kuning, <55% merah) dan pembagian 3 tier di `employees.csv`
  dirancang untuk model lama, jadi sebagian karyawan "Moderate" akan tampil hijau dengan model baru.
  Ini sengaja tidak di-tuning; bisa jadi bahan diskusi soal model drift setelah retraining.
- Folder bernama `streamlit/` memang sesuai struktur contoh, tapi jangan membuat
  `__init__.py` di dalamnya agar tidak bentrok dengan package `streamlit`.
- Performa model sengaja dibiarkan "seadanya" (ROC-AUC ~0,81) karena fokus sesi ini adalah deployment, bukan tuning.
- Cold start pertama di Vercel bisa beberapa detik (import sklearn + pandas).
- Batas ukuran bundle Vercel: jika deploy gagal karena ukuran, cek log build dan
  pastikan `.vercelignore` aktif (streamlit/, scripts/, data/ tidak ikut di-bundle).
