# Quexa — Dokumentasi Teknis

Tanggal dokumentasi: 10 Oktober 2026. Sumber: tiga ekspor flow final dan hasil pengujian yang dilaporkan pengembang. Ekspor mencatat Langflow 1.10.1.

## Masalah dan solusi

Rangkuman membantu membaca ulang materi, tetapi pengguna masih perlu menyusun soal dan mengidentifikasi konsep yang belum dipahami. Quexa membuat latihan dari PDF, memberi umpan balik dengan kutipan materi, menghitung hasil secara deterministik, lalu membuat latihan tambahan untuk topik yang dijawab salah.

MVP menjalankan pengalaman tersebut melalui IBM Bob dan Langflow. Penilaian tanpa LLM mengurangi ketidakpastian perhitungan skor, sementara kualitas soal tetap bergantung pada materi dan model.

## Arsitektur dan tanggung jawab

| Lapisan | Tanggung jawab | Batasan |
|---|---|---|
| IBM Bob | Interaksi, pemilihan tool, state kuis, pencatatan jawaban, tampilan hasil | State berada di konteks percakapan |
| MCP Streamable HTTP | Menghubungkan Bob dengan tool project Langflow | Demo memakai localhost |
| Pembuat kuis | Membaca PDF dan menghasilkan JSON soal berbasis materi | LLM, tanpa validator schema terpisah |
| Grader Python | Validasi payload, perhitungan skor, statistik topik | Menggunakan kunci yang dikirim pemanggil |
| Pembuat retry | Memfokuskan soal pada topik salah dan menghindari soal sebelumnya | Pencegahan pengulangan berbasis prompt |

Tidak ada Composio, database progres, vector database, pelatihan model, atau OCR dalam implementasi yang didokumentasikan.

## Flow 1: `quexa_grade_quiz`

Nama flow tetap mengikuti ekspor; fungsinya adalah membuat soal, bukan memberi skor.

Koneksi: Chat Input → Read File (Server File Path) → Prompt Template melalui Raw Content → Language Model → Chat Output.

Input Playground adalah teks path, misalnya:

```text
C:/Materi/rangkuman-data-warehouse.pdf
```

Prompt menerima `{materi_pdf}`. Materi diperlakukan sebagai sumber belajar, bukan instruksi. Model diminta membuat maksimal lima soal tingkat sedang, empat opsi, satu kunci, pembahasan, dan kutipan persis. Model tidak boleh menambahkan fakta di luar sumber.

Kontrak keluaran:

```json
{
  "status": "insufficient_material",
  "pesan": "Contoh ini hanya memuat satu soal.",
  "soal": [
    {
      "id": "Q1",
      "topik": "ETL",
      "pertanyaan": "Kapan transformasi dilakukan pada ETL?",
      "opsi": {
        "A": "Setelah data dimuat ke warehouse",
        "B": "Sebelum data dimuat ke warehouse",
        "C": "Setelah dashboard ditampilkan",
        "D": "Hanya setelah pengguna menjawab kuis"
      },
      "kunci": "B",
      "pembahasan": "Pada ETL, transformasi berlangsung sebelum pemuatan ke warehouse.",
      "kutipan_sumber": "Transform before data is loaded into the warehouse."
    }
  ]
}
```

Contoh di atas menunjukkan struktur kuis dengan satu soal. Pada kuis awal, `ok` mensyaratkan lima soal. Jika materi hanya mendukung satu sampai empat soal, gunakan `insufficient_material` dan pesan keterbatasan. Jika materi kosong/tidak terbaca, gunakan `unreadable_material` dan array kosong. Status adalah kontrak prompt; kesalahan pembacaan file juga dapat menghentikan flow sebelum JSON model dihasilkan.

## Flow 2: `quexa_grading`

Koneksi: Chat Input → custom `QuexaGrader` → Chat Output. Flow ini tidak memanggil LLM.

Input minimal:

```json
{
  "soal": [
    {"id": "Q1", "topik": "ETL", "kunci": "B"},
    {"id": "Q2", "topik": "ETL", "kunci": "A"}
  ],
  "jawaban": {"Q1": "b", "Q2": "D"}
}
```

Objek soal lengkap dari generator juga diterima; field tambahan tidak dipakai untuk perhitungan. Bob harus mengirim kunci dan ID asli.

Aturan validasi:

- Payload harus berupa objek JSON, `soal` array tidak kosong, dan `jawaban` objek.
- ID soal harus string tidak kosong dan unik. Jawaban untuk ID yang tidak dikenal ditolak.
- Topik harus string tidak kosong; kunci harus tepat satu huruf besar A–D.
- Jawaban dinormalisasi dengan trim dan uppercase. Nilai selain A–D ditolak.
- Jawaban hilang, `null`, atau string kosong dihitung sebagai belum dijawab.

Skor = `round(100 × benar / total_soal, 2)`. Jawaban belum diisi tetap masuk penyebut. `salah` tidak mencakup soal belum dijawab.

Keluaran untuk contoh di atas:

```json
{
  "status": "ok",
  "total_soal": 2,
  "benar": 1,
  "salah": 1,
  "belum_dijawab": 0,
  "skor": 50.0,
  "detail": [
    {"id": "Q1", "topik": "ETL", "jawaban_user": "B", "kunci": "B", "status": "benar"},
    {"id": "Q2", "topik": "ETL", "jawaban_user": "D", "kunci": "A", "status": "salah"}
  ],
  "ringkasan_topik": {
    "ETL": {"total": 2, "benar": 1, "salah": 1, "belum_dijawab": 0}
  },
  "topik_perlu_diulang": ["ETL"]
}
```

Kesalahan validasi menghasilkan `{"status":"error","pesan":"..."}`. Topik dimasukkan ke retry hanya jika memiliki jawaban salah. Topik dengan soal yang hanya dilewati tidak otomatis dimasukkan. Grader tidak memverifikasi kebenaran kunci terhadap PDF dan belum mengikat payload ke kuis tersimpan di server.

## Flow 3: `quexa_retry_quiz`

Chat Input masuk ke `QuexaRetryInput`, yang memisahkan tiga keluaran Message: path PDF, topik fokus, dan pertanyaan sebelumnya. Path diteruskan ke Read File; teks materi, topik, dan pertanyaan sebelumnya diteruskan ke Prompt Template → LLM → Chat Output.

```json
{
  "pdf_path": "C:/Materi/rangkuman-data-warehouse.pdf",
  "topik_fokus": ["ETL"],
  "soal_sebelumnya": ["Kapan transformasi dilakukan pada ETL?"]
}
```

Parser mensyaratkan path string tidak kosong, topik array string tidak kosong, dan pertanyaan sebelumnya array string. Array pertanyaan sebelumnya boleh kosong. `soal_sebelumnya` berisi teks pertanyaan, bukan objek soal atau ID.

Model diminta membuat maksimal tiga soal baru tingkat sedang untuk topik fokus, dengan ID R1–R3 dan struktur objek soal yang sama. Jika sumber tidak mendukung topik, hasil harus menjelaskan keterbatasan tanpa mengarang fakta. Status dan pesan ditentukan oleh prompt retry yang tersimpan di ekspor. Jangan memanggil flow dengan daftar topik kosong: parser menolaknya sebelum prompt berjalan.

Kuis lanjutan dinilai dengan flow grader yang sama. Skor awal dan lanjutan dilaporkan terpisah karena set dan jumlah soal dapat berbeda.

## Pembacaan PDF dan perlindungan file

Konfigurasi memakai ekstraksi teks PDF, tanpa Advanced Parser/OCR. PDF hasil scan bisa menghasilkan teks kosong. Seluruh teks hasil ekstraksi dimasukkan ke prompt; PDF besar dapat meningkatkan latensi atau melewati kapasitas input model.

Read File pada kedua flow pembuat soal memiliki patch jalur lokal pada `_validate_and_resolve_paths`:

```python
files = super()._validate_and_resolve_paths()
for file in files:
    file.delete_after_processing = False
return files
```

Patch mempertahankan PDF lokal setelah pemrosesan. Saat mengimpor ulang atau memperbarui komponen, periksa bahwa patch masih ada dan uji dua kali dengan salinan file. Path adalah lokasi di mesin Langflow, bukan otomatis di mesin pengguna atau Bob.

## Kredensial dan penanganan data

Pemeriksaan ekspor tidak menemukan API key literal; konfigurasi merujuk variabel `GOOGLE_API_KEY` dengan pemuatan dari database Langflow. Ini pemeriksaan file ekspor, bukan audit seluruh komputer, history Bob, log, atau repository.

Reviewer perlu memasukkan kredensial sendiri. Jangan menyertakan file environment atau key dalam submission. Materi dan konteks generasi dikirim ke provider model. Bob/Langflow juga dapat menyimpan history atau log; gunakan materi demo yang boleh diproses.

Endpoint localhost yang digunakan pada demo belum menunjukkan konfigurasi autentikasi untuk deployment publik. Dokumentasi ini tidak menyatakan bahwa prototipe sudah siap diakses publik.

Kunci, pembahasan, dan kutipan diterima Bob dalam hasil tool. Instruksi sesi mengatur kapan data tersebut ditampilkan; belum ada endpoint peserta yang memisahkan kunci secara aman.

## Pengujian dan bukti

| Skenario | Hasil | Jenis bukti |
|---|---|---|
| Lima jawaban benar | Skor 100, tidak ada topik retry | Dilaporkan pengembang dan diperiksa pada kode komponen |
| Lima jawaban salah | Skor 0, topik salah dirangkum | Dilaporkan pengembang dan diperiksa pada kode komponen |
| Empat benar, satu belum dijawab | Skor 80, salah 0, belum dijawab 1 | Dilaporkan pengembang dan diperiksa pada kode komponen |
| ID duplikat, ID jawaban asing, opsi E | Menghasilkan status error | Tiga pemeriksaan kode komponen |
| Payload retry valid | Path dan fokus dipisahkan dengan benar | Pemeriksaan kode komponen |
| Topik retry kosong | Ditolak sebelum generasi | Pemeriksaan kode komponen |
| Path PDF tidak valid | Error dijelaskan, tidak dibuat kuis pengganti | Dilaporkan pengembang melalui Bob |
| Topik di luar PDF | Tidak dipaksakan menjadi soal | Dilaporkan pengembang melalui Bob |
| PDF diproses berulang | File tetap tersedia | Dilaporkan pengembang melalui Bob |
| Sesi awal dan retry | Satu soal per giliran, pembahasan setelah jawaban, skor terpisah | Dilaporkan pengembang melalui Bob |

Delapan pemeriksaan kode dilakukan pada 10 Oktober 2026 dengan metode komponen Python dari ekspor dan stub antarmuka `lfx`. Pemeriksaan ini tidak menginstansiasi server Langflow, membaca PDF, memanggil provider, atau menguji MCP. Pengujian end-to-end pada tabel adalah laporan pengembang, bukan replay independen dalam penyusunan dokumentasi. Belum ada benchmark latensi, uji multiuser, atau pengukuran efektivitas belajar.

## Troubleshooting

| Gejala | Pemeriksaan |
|---|---|
| MCP disconnected | Pastikan Langflow hidup, URL memakai project hasil impor, tipe `streamable-http`, lalu refresh Bob |
| Timeout tetapi Playground berhasil | Periksa durasi model dan timeout Bob; sekitar dua menit pernah terjadi saat pengembangan |
| PDF tidak terbaca | Periksa path pada mesin Langflow, keberadaan file, izin baca, dan apakah PDF berisi teks |
| PDF hilang setelah proses | Periksa patch `delete_after_processing = False` di kedua flow |
| Model/key gagal | Isi variabel key lokal dan gunakan model yang tersedia di akun |
| JSON tidak valid | Periksa keluaran LLM di Playground; validator schema belum diterapkan |
| Retry gagal karena fokus kosong | Jangan panggil retry jika hasil grader tidak memuat topik salah |

## Checklist sebelum submission

1. Impor ekspor ke project demo dan pastikan tiga tool terlihat di Bob.
2. Jalankan satu sesi lengkap: PDF → kuis → grading → retry → grading retry.
3. Pastikan file PDF masih tersedia setelah pengujian.
4. Siapkan tangkapan layar alur Langflow, kuis Bob, skor, dan latihan lanjutan.
5. Sertakan tiga ekspor JSON, README, dan dokumentasi ini. Siapkan deck serta canvas sesuai panduan panitia.
6. Gunakan klaim sesuai bukti: prototipe berjalan pada demo lokal, belum produk teruji untuk penggunaan luas.

## Pengembangan berikutnya

Prioritas setelah demo adalah validator schema JSON, pemeriksaan kutipan terhadap teks sumber, penyimpanan kuis dan kunci di server dengan ID sesi, lalu penyimpanan progres. Dukungan OCR dan UI upload dapat ditambahkan setelah kontrak data dan pengalaman kuis stabil.
