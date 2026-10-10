# Integrasi MCP Langflow dengan IBM Bob

1. Jalankan Langflow pada `http://localhost:7860`.
2. Impor tiga JSON dalam `flows/` ke satu project.
3. Isi variabel Langflow `GOOGLE_API_KEY` menggunakan key sendiri. Periksa model yang tersedia di akun.
4. Pada pengaturan MCP project, aktifkan `quexa_grade_quiz`, `quexa_grading`, dan `quexa_retry_quiz`.
5. Salin endpoint Streamable HTTP project dari Langflow. ID project hasil impor dapat berbeda dari demo pengembang.
6. Salin `integration/mcp-config.example.json` ke `.bob/mcp.json` pada workspace Bob. Ganti `PROJECT_ID` dengan ID project lokal. Jika endpoint membutuhkan autentikasi, gunakan petunjuk konfigurasi yang ditampilkan Langflow dan simpan kredensial hanya di konfigurasi lokal.
7. Refresh MCP di Bob dan pastikan tiga tool tersedia. Nama tool dapat menyesuaikan hasil impor.
8. Jalankan flow di Playground, lalu gunakan instruksi sesi pada README untuk kuis lengkap.

Contoh konfigurasi memakai timeout 300000 ms (5 menit). Path PDF harus dapat dibaca oleh mesin Langflow. Jangan menaruh API key, token, atau konfigurasi lokal dalam commit.

## Tes koneksi

Minta Bob memanggil `quexa_grade_quiz` dengan path PDF berbasis teks yang benar-benar tersedia. Pastikan ada pemanggilan tool dan keluaran soal. Setelah kuis selesai, pastikan Bob memanggil `quexa_grading`. Uji retry dengan satu jawaban salah.

Jika koneksi gagal, periksa apakah Langflow berjalan, endpoint memakai ID project yang benar, dan tipe koneksi adalah `streamable-http`. Jangan membuka endpoint localhost ke publik sebagai bagian dari setup demo ini.
