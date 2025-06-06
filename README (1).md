# 🐖 pdtbank (Contoh Proyek UAP) 
Proyek ini merupakan sistem perbankan sederhana yang dibangun menggunakan PHP dan MySQL. Tujuannya adalah untuk mengelola transaksi keuangan secara aman dan konsisten, dengan memanfaatkan stored procedure, trigger, transaction, dan function. Sistem ini juga dilengkapi mekanisme backup otomatis untuk menjaga keamanan data jika terjadi hal yang tidak diinginkan.

![Home](assets/img/home.png)

## 📌 Detail Konsep

### ⚠️ Disclaimer

Peran  **stored procedure**, **trigger**, **transaction**, dan **function** dalam proyek ini dirancang khusus untuk kebutuhan sistem **pdtbank**. Penerapannya bisa berbeda pada sistem lain, tergantung arsitektur dan kebutuhan masing-masing sistem.

### 🧠 Stored Procedure 
Stored procedure bertindak seperti SOP internal yang menetapkan alur eksekusi berbagai operasi penting di sistem perbankan. Procedure ini disimpan langsung di lapisan database, sehingga dapat menjamin konsistensi, efisiensi, dan keamanan eksekusi, terutama dalam sistem terdistribusi atau multi-user.

![Procedure](assets/img/procedure.png)

Beberapa procedure penting yang digunakan:
* **```deposit_money(p_transaction_id, p_to_account, p_amount)```**: Menambah saldo akun pengguna serta mencatat detail transaksi setoran.
  ```php
  // Call the deposit_money stored procedure
  $stmt = $this->conn->prepare("CALL deposit_money(?, ?, ?)");
  $stmt->execute([
      $txId,
      $toAccount['account_number'],
      $amount
  ]);
  ```
* **```transfer_money(p_transaction_id, p_from_account, p_to_account, p_amount)```**: Memastikan saldo pengirim cukup, memperbarui saldo kedua pihak, dan mencatat detail transaksi.
  ```php
  // Call the transfer_money stored procedure
  $stmt = $this->conn->prepare("CALL transfer_money(?, ?, ?, ?)");
  $stmt->execute([
      $txId,
      $fromAccount['account_number'],
      $toAccountNumber,
      $amount
  ]);
  ```
* **```get_transaction_history(account)```**: Mengambil daftar riwayat transaksi akun pengguna.
  ```php
  // Call the get_transaction_history stored procedure
  $stmt = $this->conn->prepare("CALL get_transaction_history(?)");
  $stmt->execute([$accountNumber]);
  $raw = $stmt->fetchAll(PDO::FETCH_ASSOC);
  ```
Dengan menyimpan proses-proses ini di sisi database, sistem menjaga integritas data di level paling dasar, terlepas dari cara aplikasi mengaksesnya.

### 🚨 Trigger
Trigger `validate_transaction` berfungsi sebagai sistem pengaman otomatis yang aktif sebelum data masuk ke dalam tabel. Seperti palang pintu yang hanya terbuka jika syarat tertentu terpenuhi, trigger mencegah input data yang tidak valid atau berisiko merusak integritas sistem.

![Trigger](assets/img/trigger.png)

Trigger `validate_transaction` otomatis aktif pada procedure berikut:
* ```transfer_money```
```sql
INSERT INTO transactions (transaction_id, from_account, to_account, amount)
VALUES (p_transaction_id, p_from_account, p_to_account, p_amount);
```
* ```deposit_money```
```sql
INSERT INTO transactions (transaction_id, from_account, to_account, amount)
VALUES (p_transaction_id, 'Cash Deposit ATM', p_to_account, p_amount);
```

Beberapa peran trigger di sistem ini:
* Menolak transaksi ke akun yang invalid.
* Menolak transaksi dengan jumlah tidak logis (≤ 0).
* Mencegah duplikasi transaksi dengan `transaction_id` yang sama.

Dengan adanya trigger di lapisan database, validasi tetap dijalankan secara otomatis, bahkan jika ada celah atau kelalaian dari sisi aplikasi. Ini selaras dengan prinsip reliabilitas pada sistem terdistribusi.

### 🔄 Transaction (Transaksi)
Dalam sistem perbankan, sebuah transaksi seperti transfer atau pembukaan rekening tidak dianggap berhasil jika hanya sebagian prosesnya yang selesai. Semua langkah harus dijalankan hingga tuntas — jika salah satu gagal, seluruh proses dibatalkan. Prinsip ini diwujudkan melalui penggunaan `beginTransaction()` dan `commit()` di PHP.

Contohnya, pada proses transfer dan deposit, sistem akan memulai transaksi, menjalankan procedure penyimpanan (stored procedure), lalu meng-commit perubahan jika berhasil. Namun, jika ditemukan masalah — seperti saldo tidak mencukupi atau akun tidak ditemukan — maka seluruh proses dibatalkan menggunakan `rollback()`. Hal ini mencegah perubahan data yang parsial, seperti saldo yang terpotong padahal transaksi tidak sah.

```App\Models\Transaction.php```
* Implementasi transaction untuk procedure ```deposit_money```
  ```php
  try {
      // Start a transaction
      // This is important to ensure that the deposit is atomic
      $this->conn->beginTransaction();
      // Call the deposit_money stored procedure
      $stmt = $this->conn->prepare("CALL deposit_money(?, ?, ?)");
      $stmt->execute([
          $txId,
          $toAccount['account_number'],
          $amount
      ]);

      $this->conn->commit();
  } catch (PDOException $e) {
      $this->conn->rollBack();
      $errorInfo = $e->errorInfo ?? [];
      $message = $errorInfo[2] ?? $e->getMessage();

      throw new Exception("Deposit failed: SQLSTATE[{$errorInfo[0]}]: {$errorInfo[1]} {$message}");
        }
* Implementasi transaction untuk procedure ```transfer_money```
  ```php
  try {
      // Start a transaction
      // This is important to ensure that the transfer is atomic
      $this->conn->beginTransaction();
      // Call the transfer_money stored procedure
      $stmt = $this->conn->prepare("CALL transfer_money(?, ?, ?, ?)");
      $stmt->execute([
          $txId,
          $fromAccount['account_number'],
          $toAccountNumber,
          $amount
      ]);

      $this->conn->commit();
  } catch (PDOException $e) {
      $this->conn->rollBack();
      $errorInfo = $e->errorInfo ?? [];
      $message = $errorInfo[2] ?? $e->getMessage();

      throw new Exception("Transfer failed: SQLSTATE[{$errorInfo[0]}]: {$errorInfo[1]} {$message}");
    }
  ```

Demikian pula saat user melakukan registrasi, sistem tidak hanya menyimpan data user, tetapi juga membuat akun bank sekaligus. Proses ini dijalankan dalam satu transaksi agar semua langkah saling bergantung dan terjamin konsistensinya.

```App\Models\User.php```
```php
try {
    // Start a transaction
    // This is important to ensure that the registration is atomic
    $this->conn->beginTransaction();

    $stmt = $this->conn->prepare("INSERT INTO users (username, password) VALUES (?, ?)");
    $stmt->execute([$username, $hashedPassword]);
    $userId = $this->conn->lastInsertId();

    $accountNumber = $this->generateUniqueAccountNumber();
    $stmtAcc = $this->conn->prepare("INSERT INTO accounts (user_id, account_number, balance) VALUES (?, ?, 0)");
    $stmtAcc->execute([$userId, $accountNumber]);

    $this->conn->commit();
    return true;
} catch (Exception $e) {
    $this->conn->rollBack();
    throw new Exception("Registration failed due to database error.");
}
```
### 📺 Function 
Function digunakan untuk mengambil informasi tanpa mengubah data. Seperti layar monitor: hanya menampilkan data, tidak mengubah apapun.

Contohnya, fungsi  `get_balance(account)` mengembalikan saldo terkini dari sebuah akun. 

Fungsi ini dipanggil baik dari aplikasi maupun dari procedure yang ada di database. Dengan begitu, logika pembacaan saldo tetap terpusat dan konsisten, tanpa perlu duplikasi kode atau resiko ketidaksesuaian antara sistem aplikasi dan database.
* Aplikasi

  ``App/Models/Account.php``
  ```php
  $stmt = $this->conn->prepare("SELECT get_balance(?) AS balance");
  $stmt->execute([$accountNumber]);
  $result = $stmt->fetch(PDO::FETCH_ASSOC);
  ```
* Procedure ```transfer_money```
  ```sql
  SET v_balance = get_balance(p_from_account);
  ```
  
![Function](assets/img/function.png)

Penggunaan function seperti ini mencerminkan praktik pemisahan logika bisnis di database layer, yang relevan dalam konteks Pemrosesan Data Terdistribusi — di mana konsistensi dan reliabilitas antar node atau proses sangat krusial.

### 🔄 Backup Otomatis
Untuk menjaga ketersediaan dan keamanan data, sistem dilengkapi fitur backup otomatis menggunakan `mysqldump`dan task scheduler. Backup dilakukan secara berkala dan disimpan dengan nama file yang mencakup timestamp, sehingga mudah ditelusuri. Semua file disimpan di direktori `storage/backups`.

`backup.php`
```php
<?php
require_once __DIR__ . '/init.php';

$date = date('Y-m-d_H-i-s');
$backupFile = __DIR__ . "/storage/backups/pdtbank_backup_$date.sql";
$command = "\"C:\\laragon\\bin\\mysql\\mysql-8.0.30-winx64\\bin\\mysqldump.exe\" -u " . DB_USER . " " . DB_NAME . " > \"$backupFile\"";
exec($command);
```

## 🧩 Relevansi Proyek dengan Pemrosesan Data Terdistribusi
Sistem ini dirancang dengan memperhatikan prinsip-prinsip dasar pemrosesan data terdistribusi:
* **Konsistensi**: Semua transaksi dieksekusi dengan procedure dan validasi terpusat di database.
* **Reliabilitas**: Trigger dan transaction memastikan sistem tetap aman meskipun ada error atau interupsi.
* **Integritas**: Dengan logika disimpan di dalam database, sistem tetap valid walaupun dipanggil dari banyak sumber (web, API, dsb).


