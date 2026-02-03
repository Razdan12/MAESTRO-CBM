# 📘 BLUEPRINT SOFTWARE: MAESTRO CBM

**Sistem Inspeksi Otomatis Rolling Stock Berbasis AI**

## 1. Arsitektur Sistem (System Architecture)

Sistem ini menggunakan arsitektur **Hybrid Edge-Cloud**. Pemrosesan berat yang sensitif terhadap waktu dilakukan di pinggir rel (*Edge*), sementara manajemen data dan pelaporan dilakukan di Server Pusat (*Cloud/On-Premise*).

### A. Topologi Fisik & Data

1. **Edge Node (Location Case):**
* Berisi PC Industri dengan GPU.
* Terhubung langsung ke Kamera Baumer (via Gigabit Ethernet/USB3) dan Sensor (via I/O module).
* Tugas: Akuisisi gambar, sinkronisasi sensor, *pre-processing*, dan inferensi AI awal (opsional).


2. **Central Server:**
* Tugas: Penyimpanan jangka panjang, pelatihan ulang model AI, *web dashboard hosting*, dan API gateway untuk aplikasi mobile.



---

## 2. Alur Kerja Software (Software Flow)

Mengacu pada diagram arsitektur, berikut adalah perjalanan data dari sensor hingga ke pengguna:

### Langkah 1: Pemicu & Akuisisi (Triggering)

* **Input:** Sensor *Axle Counter* mendeteksi roda kereta.
* **Proses:**
1. Signal masuk ke `TriggerService`.
2. `TriggerService` mengirim perintah *capture* ke semua kamera secara simultan (sinkron).
3. Kamera mengambil gambar *high-speed* dan mengirim data mentah (RAW) ke memori PC.



### Langkah 2: Pengolahan Gambar (Pre-processing)

* **Input:** Gambar RAW dari kamera.
* **Proses:**
1. **Image Cleaning:** Mengurangi *noise* akibat debu atau hujan.
2. **Auto-Cropping (ROI):** Memotong gambar menjadi bagian-bagian kecil spesifik (misal: hanya kotak "Baut Roda" atau "Pegas").
3. **Metadata Tagging:** Menambahkan *timestamp*, ID Kamera, dan urutan gandar pada header gambar.



### Langkah 3: Analisis AI (Intelligence)

* **Input:** Potongan gambar (ROI).
* **Proses (Parallel):**
* **Pipeline 1 (OCR):** Membaca Nomor Gerbong pada bodi kereta untuk identifikasi aset.


* **Pipeline 2 (Object Detection):** Mendeteksi keberadaan komponen (Baut, Pegas, Buffer).
* **Pipeline 3 (Anomaly Detection):**
* Cek Marker Baut: Apakah garis putus/bergeser?.
* Cek Stiker Suhu: Apakah warna berubah (indikasi panas)?.
* Cek Fisik: Apakah pegas patah atau selang terlepas?.



### Langkah 4: Pengambilan Keputusan & Pelaporan

* **Input:** Hasil skor probabilitas dari AI.
* **Proses:**
1. **Rule Engine:** Jika `probabilitas_kerusakan > 85%`, tandai sebagai "Anomali".
2. **Aggregation:** Menggabungkan status semua komponen untuk menentukan status akhir Gerbong (GO / NO-GO).
3. **Alerting:** Kirim notifikasi ke HP Maintainer via Firebase/APNS.





---

## 3. Fitur Inti (Core Features)

### A. Fitur Deteksi (AI Capabilities)

1. **Wheel Bolt Monitor:** Deteksi kelonggaran baut roda melalui pergeseran garis penanda (*misalignment marker*).
2. **Thermal Indicator Check:** Deteksi *overheat* pada *bearing* melalui perubahan warna stiker suhu.
3. **Mechanical Structural Check:** Deteksi fisik pegas (*spring*), *buffer plate*, dan selang angin (*brake hose*).
4. **Automatic Train Identification:** Pencatatan nomor gerbong otomatis via OCR.



### B. Fitur Aplikasi (User Interface)

1. **Real-time Alert System:** Notifikasi *pop-up* di HP petugas saat anomali ditemukan.
2. **Digital Inspection Report:** Generasi laporan PDF otomatis menggantikan formulir kertas.
3. **Verification Module:** Fitur bagi petugas untuk memvalidasi temuan AI (Benar/Salah) untuk umpan balik perbaikan model.
4. **Dashboard Analytics:** Statistik tren kerusakan dan performa armada.

---

## 4. Rekomendasi Teknologi (Tech Stack)

| Lapisan | Teknologi | Alasan |
| --- | --- | --- |
| **Backend & AI** | **Python 3.9+** | Standar industri AI. Pustaka lengkap. |
| **AI Framework** | **PyTorch / Ultralytics YOLOv8** | Performa *real-time* terbaik untuk deteksi objek kecil. |
| **Computer Vision** | **OpenCV & NVIDIA DALI** | Pengolahan gambar super cepat di GPU. |
| **Database** | **PostgreSQL (Data) + MinIO (Images)** | SQL untuk data relasional, MinIO untuk penyimpanan *blob* gambar efisien. |
| **API Framework** | **FastAPI** | Sangat cepat (asynchronous), cocok untuk I/O kamera & AI. |
| **Frontend/Mobile** | **React Native / Flutter** | Satu kode untuk Android & iOS (Maintainer App). |
| **Message Broker** | **Redis / MQTT** | Antrean data sensor *real-time* agar tidak macet. |
| **Hardware Driver** | **Baumer GAPI SDK (Python Wrapper)** | SDK resmi untuk kontrol kamera Baumer.

 |

---

## 5. Struktur Kode (Project Structure)

Berikut adalah usulan struktur folder untuk menjaga kode tetap rapi dan *scalable*.

```text
maestro-cbm/
├── app/
│   ├── api/                # Endpoint API (REST)
│   │   ├── endpoints/      # Definisikan route (misal: /inspection, /alert)
│   │   └── deps.py         # Dependencies (DB session, Auth)
│   ├── core/               # Konfigurasi utama
│   │   ├── config.py       # Environment variables
│   │   └── security.py     # JWT Token & Authentication
│   ├── ai_engine/          # Otak Kecerdasan Buatan
│   │   ├── models/         # File model (.pt, .onnx)
│   │   ├── inference.py    # Logika deteksi (YOLO/PyTorch)
│   │   └── preprocessing.py# Crop, Resize, Enhance gambar
│   ├── drivers/            # Koneksi ke Hardware
│   │   ├── camera_baumer.py # Kontrol kamera (Trigger, Exposure)
│   │   └── sensors.py      # Baca Axle Counter & Suhu
│   ├── services/           # Logika Bisnis
│   │   ├── inspection.py   # Alur inspeksi & agregasi hasil
│   │   └── notification.py # Kirim alert ke Mobile
│   └── schemas/            # Pydantic models (Validasi data)
├── db/                     # Database
│   ├── base.py
│   └── models.py           # Definisi tabel (SQLAlchemy)
├── tests/                  # Unit Testing
├── scripts/                # Script untuk start/setup
├── requirements.txt        # Daftar library Python
└── Dockerfile              # Konfigurasi container deploy

```

## 6. Contoh Logika Koding (Pseudocode)

**Modul: AI Inference (Deteksi Baut)**

```python
# app/ai_engine/inference.py

from ultralytics import YOLO
import cv2

# Load model yang sudah dilatih
model_baut = YOLO('models/best_baut_v1.pt')

def analisa_baut(image_path):
    """
    Menganalisa gambar roda untuk mencari baut kendor.
    """
    # 1. Baca gambar
    img = cv2.imread(image_path)
    
    # 2. Lakukan deteksi
    results = model_baut.predict(img, conf=0.8) # Threshold 80%
    
    anomalies = []
    
    for result in results:
        for box in result.boxes:
            class_id = int(box.cls)
            label = model_baut.names[class_id]
            
            # Jika terdeteksi kelas "misalignment" (baut geser)
            if label == 'misaligned_bolt':
                anomalies.append({
                    "type": "BAUT_KENDOR",
                    "confidence": float(box.conf),
                    "location": box.xyxy.tolist()
                })
                
    return anomalies

```

Blueprint ini sudah mencakup seluruh aspek teknis yang diperlukan tim pengembang untuk memulai proyek MAESTRO CBM dari nol hingga implementasi.
