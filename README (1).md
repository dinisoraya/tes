# 🐖 pdtbank (Contoh Proyek UAP) 
Proyek ini merupakan sistem perbankan sederhana yang dibangun menggunakan PHP dan MySQL. Tujuannya adalah untuk mengelola transaksi keuangan secara aman dan konsisten, dengan memanfaatkan stored procedure, trigger, transaction, dan function. Sistem ini juga dilengkapi mekanisme backup otomatis untuk menjaga keamanan data jika terjadi hal yang tidak diinginkan.

![Home](assets/img/home.png)

## 📌 Detail Konsep

### ⚠️ Disclaimer

Peran  **stored procedure**, **trigger**, **transaction**, dan **function** dalam proyek ini dirancang khusus untuk kebutuhan sistem **pdtbank**. Penerapannya bisa berbeda pada sistem lain, tergantung arsitektur dan kebutuhan masing-masing sistem.

### 🧠 Stored Procedure 
Stored procedure bertindak seperti SOP internal yang menetapkan alur eksekusi berbagai operasi penting di sistem perbankan. Prosedur-prosedur ini disimpan langsung di lapisan database, sehingga dapat menjamin konsistensi, efisiensi, dan keamanan eksekusi, terutama dalam sistem terdistribusi atau multi-user.

![Procedure](assets/img/procedure.png)

Beberapa prosedur penting yang digunakan:
* **deposit_money(p_transaction_id, p_to_account, p_amount)**: Menambah saldo akun pengguna serta mencatat detail transaksi setoran.
* **transfer_money(p_transaction_id, p_from_account, p_to_account, p_amount)**: Memastikan saldo pengirim cukup, memperbarui saldo kedua pihak, dan mencatat detail transaksi.
* **get_transaction_history(account)**: Mengambil daftar riwayat transaksi akun pengguna.

Dengan menyimpan proses-proses ini di sisi database, sistem menjaga integritas data di level paling dasar, terlepas dari cara aplikasi mengaksesnya.

### 🚨 Trigger
Trigger `validate_transaction` berfungsi sebagai sistem pengaman otomatis yang aktif sebelum data masuk ke dalam tabel. Seperti palang pintu yang hanya terbuka jika syarat tertentu terpenuhi, trigger mencegah input data yang tidak valid atau berisiko merusak integritas sistem.

![Trigger](assets/img/trigger.png)

Beberapa peran trigger di sistem ini:
* Menolak transaksi ke akun yang invalid.
* Menolak transaksi dengan jumlah tidak logis (≤ 0).
* Mencegah duplikasi transaksi dengan `transaction_id` yang sama.

Dengan adanya trigger di lapisan database, validasi tetap dijalankan secara otomatis, bahkan jika ada celah atau kelalaian dari sisi aplikasi. Ini selaras dengan prinsip reliabilitas pada sistem terdistribusi.

### 🔄 Transaction (Transaksi)
Dalam sistem perbankan, sebuah transaksi seperti transfer atau pembukaan rekening tidak dianggap berhasil jika hanya sebagian prosesnya yang selesai. Semua langkah harus dijalankan hingga tuntas — jika salah satu gagal, seluruh proses dibatalkan. Prinsip ini diwujudkan melalui penggunaan `beginTransaction()` dan `commit()` di PHP.

Contohnya, pada proses transfer dan deposit, sistem akan memulai transaksi, menjalankan prosedur penyimpanan (stored procedure), lalu meng-commit perubahan jika berhasil. Namun, jika ditemukan masalah — seperti saldo tidak mencukupi atau akun tidak ditemukan — maka seluruh proses dibatalkan menggunakan `rollback()`. Hal ini mencegah perubahan data yang parsial, seperti saldo yang terpotong padahal transaksi tidak sah.

Demikian pula saat user melakukan registrasi, sistem tidak hanya menyimpan data user, tetapi juga membuat akun bank sekaligus. Proses ini dijalankan dalam satu transaksi agar semua langkah saling bergantung dan terjamin konsistensinya.

### 📺 Function 
Function digunakan untuk mengambil informasi tanpa mengubah data. Seperti layar monitor: hanya menampilkan data, tidak mengubah apapun.

Contohnya, fungsi  `get_balance(account)` mengembalikan saldo terkini dari sebuah akun. 

Fungsi ini dipanggil baik dari aplikasi maupun dari prosedur yang ada di database. Dengan begitu, logika pembacaan saldo tetap terpusat dan konsisten, tanpa perlu duplikasi kode atau resiko ketidaksesuaian antara sistem aplikasi dan database.

![Function](assets/img/function.png)

Penggunaan function seperti ini mencerminkan praktik pemisahan logika bisnis di database layer, yang relevan dalam konteks Pemrosesan Data Terdistribusi — di mana konsistensi dan reliabilitas antar node atau proses sangat krusial.

### 🔄 Backup Otomatis
Untuk menjaga ketersediaan dan keamanan data, sistem dilengkapi fitur backup otomatis menggunakan `mysqldump`dan task scheduler. Backup dilakukan secara berkala dan disimpan dengan nama file yang mencakup timestamp, sehingga mudah ditelusuri. Semua file disimpan di direktori `storage/backups`.
## 🧩 Relevansi Proyek dengan Pemrosesan Data Terdistribusi
Sistem ini dirancang dengan memperhatikan prinsip-prinsip dasar pemrosesan data terdistribusi:
* **Konsistensi**: Semua transaksi dieksekusi dengan prosedur dan validasi terpusat di database.
* **Reliabilitas**: Trigger dan transaction memastikan sistem tetap aman meskipun ada error atau interupsi.
* **Integritas**: Dengan logika disimpan di dalam database, sistem tetap valid walaupun dipanggil dari banyak sumber (web, API, dsb).


