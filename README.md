# Quexa

**Ubah rangkuman PDF menjadi latihan ujian, lalu latih kembali topik yang masih salah.**

Quexa adalah prototipe agent belajar untuk hackathon. IBM Bob menjalankan percakapan kuis, Langflow mengolah materi dan membuat soal, sedangkan komponen Python menghitung skor tanpa LLM.

## Alur penggunaan

1. Pengguna memberikan lokasi PDF materi pada komputer yang menjalankan Langflow.
2. Quexa membuat maksimal lima soal pilihan ganda tingkat sedang berdasarkan teks PDF.
3. Bob menampilkan satu soal per giliran dan mencatat jawaban pengguna.
4. Setelah pengguna menjawab, Bob menampilkan kunci, pembahasan, dan kutipan pendukung dari materi.
5. Setelah kuis selesai, grader menghitung skor dan merangkum hasil per topik.
6. Pengguna dapat melanjutkan dengan maksimal tiga soal baru untuk topik yang dijawab salah.

## Komponen proyek

| File ekspor | Fungsi | Input |
|---|---|---|
| `quexa_grade_quiz.json` | Membuat kuis awal; nama flow ini tetap mengikuti ekspor | Path PDF sebagai teks |
| `quexa_grading.json` | Menghitung skor dan topik yang perlu diulang | JSON soal dan jawaban |
| `quexa_retry_quiz.json` | Membuat latihan lanjutan | JSON path PDF, topik fokus, dan pertanyaan sebelumnya |

Ketiga file ekspor tersedia di [flows/](flows/). Rincian kontrak input dan pengujian ada di [dokumentasi teknis](docs/TECHNICAL_DOCUMENTATION.md). Panduan integrasi tersedia di [MCP Setup](docs/MCP_SETUP.md).

## Lingkungan yang digunakan

- Windows, IBM Bob, dan Langflow pada komputer yang sama.
- Ekspor mencatat `last_tested_version: 1.10.1` untuk Langflow.
- Bob versi 2.2.1 terlihat pada tangkapan layar pengembangan; bukan persyaratan kompatibilitas yang telah diuji lintas versi.
- Model pada ekspor: `gemini-3.8-flash`, provider Google Generative AI. Nama ini merupakan konfigurasi yang tersimpan, bukan jaminan ketersediaan model pada akun reviewer.
- API key disimpan melalui variabel Langflow `GOOGLE_API_KEY`; reviewer menggunakan kredensial sendiri.
- Integrasi Bob–Langflow menggunakan MCP Streamable HTTP. Composio tidak digunakan.

## Setup demo lokal

1. Jalankan Langflow dan buka antarmukanya pada `http://localhost:7860`.
2. Impor tiga ekspor flow ke satu project Langflow.
3. Isi variabel `GOOGLE_API_KEY` di lingkungan Langflow. Jika model pada ekspor tidak tersedia, pilih model yang tersedia di akun dan uji ulang keluaran JSON.
4. Pada flow pembuat kuis dan retry, periksa koneksi Read File → Raw Content → Prompt Template. Gunakan PDF dengan teks yang bisa diekstrak.
5. Pertahankan patch Read File yang mengatur `delete_after_processing = False` untuk file lokal. Uji dengan salinan PDF dan pastikan file masih ada setelah flow berjalan.
6. Ambil endpoint MCP Streamable HTTP project dari Langflow. Ganti `PROJECT_ID` pada konfigurasi Bob berikut dengan ID project hasil impor.

```json
{
  "mcpServers": {
    "lf-quexa": {
      "type": "streamable-http",
      "url": "http://localhost:7860/api/v1/mcp/project/PROJECT_ID/streamable",
      "disabled": false
    }
  }
}
```

7. Masukkan konfigurasi ke konfigurasi MCP Bob (`.bob/mcp.json` pada workspace yang digunakan), refresh koneksi, dan aktifkan tiga tool flow Quexa. Gunakan nama tool yang benar-benar muncul setelah impor.
8. Contoh konfigurasi di [integration/mcp-config.example.json](integration/mcp-config.example.json) menggunakan timeout 300000 ms (5 menit), sesuai konfigurasi demo. Sesuaikan jika diperlukan.
9. Jalankan setiap flow di Playground sebelum menjalankan seluruh sesi lewat Bob.

Konfigurasi ini untuk demo pada komputer yang sama. PDF path merujuk filesystem mesin Langflow. Jika Langflow berjalan dalam container atau mesin lain, file harus tersedia pada filesystem tersebut dan path disesuaikan.

## Instruksi sesi untuk Bob

Tempel instruksi berikut ke sesi Bob setelah tiga tool tersedia:

```text
Kamu adalah Quexa, pendamping latihan ujian berbasis materi pengguna.
Gunakan tiga tool Langflow yang tersedia:
- quexa_grade_quiz untuk membuat kuis awal;
- quexa_grading untuk menghitung skor;
- quexa_retry_quiz untuk membuat latihan lanjutan.
Sesuaikan pemanggilan dengan nama dan skema tool yang ditampilkan MCP.

Minta path PDF jika belum tersedia. Simpan path, JSON kuis, pertanyaan
yang sudah digunakan, dan jawaban dalam konteks sesi.
Periksa status hasil tool. Jika materi tidak terbaca atau tidak mendukung
soal, jelaskan pesan yang diterima. Jangan mengarang kuis pengganti.
Jika hanya beberapa soal tersedia, gunakan soal yang tersedia.

Tampilkan satu soal per giliran: ID, pertanyaan, dan opsi A–D.
Jangan tampilkan kunci, pembahasan, atau kutipan sebelum pengguna menjawab.
Terima A–D tanpa membedakan huruf besar/kecil; minta perbaikan untuk input
lain. Jika pengguna memilih lewati, simpan null.
Setelah jawaban diterima, tampilkan kunci, pembahasan, dan kutipan dari
JSON soal. Jangan mengubah kunci atau mengganti isi materi.

Setelah semua soal selesai, panggil quexa_grading dengan objek berisi
soal asli dan jawaban yang dipetakan berdasarkan ID. Gunakan hasil grader
untuk skor, jumlah benar/salah/belum dijawab, dan ringkasan per topik.

Jika topik_perlu_diulang tidak kosong, tawarkan latihan lanjutan.
Jika pengguna setuju, panggil quexa_retry_quiz dengan pdf_path,
topik_fokus dari hasil grader, serta soal_sebelumnya berupa array teks
pertanyaan yang sudah digunakan. Jalankan kuis lanjutan satu soal per
giliran dan nilai dengan quexa_grading. Laporkan skor awal dan lanjutan
secara terpisah. Jika tidak ada topik salah, jangan panggil retry.

Jika tool gagal atau timeout, jelaskan kegagalan dan jangan menyatakan
hasil berhasil. Jangan menampilkan konfigurasi kredensial atau API key.
```

## Contoh mulai demo

```text
Mulai latihan Quexa dari PDF C:/Materi/rangkuman-data-warehouse.pdf.
Tampilkan satu soal per giliran. Setelah selesai, nilai jawaban saya
dan tawarkan latihan lanjutan untuk topik yang masih salah.
```

Gunakan file yang benar-benar ada. Untuk demo, siapkan PDF pendek dengan beberapa konsep berbeda, jawab beberapa soal salah, lalu tunjukkan latihan lanjutan.

## Batasan MVP

- Input berupa path PDF lokal, belum berupa UI upload file. PDF hasil scan belum didukung melalui OCR pada konfigurasi ini.
- Soal dibuat dari teks materi yang dimasukkan ke prompt; belum ada retrieval berbasis vector database.
- Validitas JSON, ketepatan kutipan, dan pencegahan soal berulang masih mengandalkan instruksi model.
- Kunci tersedia dalam keluaran tool ke Bob. Penyembunyian kunci sebelum menjawab adalah perilaku percakapan, belum pemisahan data aman di server.
- State sesi berada dalam percakapan Bob; belum ada penyimpanan progres permanen atau akun pengguna.
- Materi dikirim ke provider model untuk generasi. Gunakan materi yang boleh diproses melalui layanan tersebut.
- Skor menunjukkan hasil pada soal yang dihasilkan; belum ada studi peningkatan hasil belajar atau jaminan cakupan ujian.

## Status pengujian

Pengembang melaporkan kuis awal, grading, retry, dan pemeliharaan file PDF berhasil melalui Bob. Pemeriksaan terpisah pada kode ekspor meluluskan delapan pemeriksaan komponen Python menggunakan stub antarmuka `lfx`; pemeriksaan ini tidak menjalankan server Langflow atau koneksi MCP. Detail dan checklist demo ada pada dokumentasi teknis.

## Isi repository

- [Flow Langflow](flows/)
- [Dokumentasi teknis](docs/TECHNICAL_DOCUMENTATION.md)
- [Panduan MCP Bob–Langflow](docs/MCP_SETUP.md)
- [Contoh konfigurasi MCP](integration/mcp-config.example.json)
- [Contoh percakapan](prompts/contoh%20penggunaan.md)
- [Screenshot demo dan pengujian](docs/screenshots/)

PDF pada `sample/` adalah materi contoh yang digunakan saat pengembangan, bukan karya yang diklaim dibuat oleh proyek Quexa. Hak materi tetap pada pemiliknya; distribusi publiknya perlu mendapat izin yang sesuai.
