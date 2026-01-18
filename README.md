# install odbc linux ubuntu22.04 pada aapanel

# PHP 5.6 → SQL Server 2014 / 2017 / 2022
## via ODBC (FreeTDS + unixODBC)

## 🎯 Target Akhir
- PHP 5.6+
- Driver: PDO + ODBC
- DSN:
```
$dsn = "odbc:sql2017";
```

## SQL Server:
- ✅ 2014
- ✅ 2017
- ✅ 2022

## 🧩 Arsitektur Koneksi
```
PHP 5.6
 └─ PDO
    └─ pdo_odbc
       └─ unixODBC
          └─ FreeTDS
             └─ SQL Server
```

## 1️⃣ Install dependency OS (WAJIB)
```
sudo apt update
sudo apt install -y \
  freetds-bin \
  freetds-dev \
  unixodbc \
  unixodbc-dev \
  odbcinst
```

### Cek FreeTDS:
```
tsql -C
```

### Harus ada:
```
Version: freetds v1.3.x
unixodbc: yes
```

## 2️⃣ Konfigurasi FreeTDS
📍 /etc/freetds/freetds.conf
```
[global]
    tds version = 7.4

[sql2017]
    host = 127.0.0.1
    port = 1433
    tds version = 7.4
```

#### ℹ️ tds version 7.4 aman untuk:
- SQL Server 2014
- SQL Server 2017
- SQL Server 2022

### Test FreeTDS langsung
```
tsql -S sql2017 -U sa -P 123456
```

#### Kalau dapat prompt:
```
1>
```
✅ LANJUT

## 3️⃣ Buat ODBC Driver FreeTDS

📍 `/etc/odbcinst.ini`
```
[FreeTDS]
Description = FreeTDS Driver
Driver      = /usr/lib/x86_64-linux-gnu/odbc/libtdsodbc.so
Setup       = /usr/lib/x86_64-linux-gnu/odbc/libtdsS.so
UsageCount  = 1
```

### Pastikan file ada:
```
ls -l /usr/lib/x86_64-linux-gnu/odbc/libtdsodbc.so
```

### Jika tidak ada:
```
dpkg -L freetds-dev | grep odbc
```

## 4️⃣ Buat DSN ODBC

📍 `/etc/odbc.ini`
```
[sql2017]
Description = SQL Server via FreeTDS
Driver      = FreeTDS
Servername  = sql2017
Database    = Surabaya_System
Port        = 1433
TDS_Version = 7.4
```

## 5️⃣ Test ODBC (LEVEL SYSTEM)
```
isql -v sql2017 saw 123456
```

### Jika berhasil:
```
SQL>
```
#### 👉 WAJIB lolos di sini
Kalau isql gagal → PHP pasti gagal

## 6️⃣ Aktifkan pdo_odbc di PHP 5.6 (aaPanel)
6.1 Cek dulu

```
/www/server/php/56/bin/php -m | grep odbc
```
Kalau belum ada → lanjut compile

### 6.2 Compile pdo_odbc (AMAN & BENAR)
```
cd /usr/local/src
wget https://www.php.net/distributions/php-5.6.40.tar.gz
tar zxvf php-5.6.40.tar.gz
cd php-5.6.40/ext/pdo_odbc
```

### Build pakai php-config aaPanel:
```
/www/server/php/56/bin/phpize

./configure \
 --with-php-config=/www/server/php/56/bin/php-config \
 --with-pdo-odbc=unixODBC,/usr

make
make install
```
### 6.3 Enable extension

📍 `/www/server/php/56/etc/php.ini`
```
extension=pdo.so
extension=pdo_odbc.so
```

### Restart PHP:
```
service php-fpm-56 restart
```

### Cek:
```
/www/server/php/56/bin/php -m | grep odbc
```

### Harus muncul:
```
PDO
PDO_ODBC
odbc
```

## 7️⃣ Test PHP (FINAL)
```
<?php
$dsn  = "odbc:sql2017";
$user = "sa";
$pass = "123456";

try {
    $pdo = new PDO($dsn, $user, $pass);
    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);

    echo "✅ CONNECTED via ODBC + FreeTDS (PHP 5.6)<br>";

    $stmt = $pdo->query("SELECT TOP 5 name FROM sys.tables");
    foreach ($stmt as $row) {
        echo $row['name'] . "<br>";
    }

} catch (PDOException $e) {
    echo "❌ FAILED: " . $e->getMessage();
}
```

### 🧪 Kompatibilitas SQL Server
| SQL Server	| Status	| Catatan |
| :--- | :---: | :--- |
| 2014	| ✅	| TDS 7.3 / 7.4 |
| 2017	| ✅	| 7.4 recommended |
| 2022	| ✅	|Encryption OFF / legacy |

### Jika SQL 2022:
```
ALTER LOGIN saw WITH CHECK_POLICY = OFF;
```

#### Dan pastikan:
- SQL Server → Allow legacy TLS
- Firewall buka port 1433

## ❗ Error & Solusi Cepat
### ❌ could not find driver
→ pdo_odbc.so belum aktif

## ❌ Adaptive Server connection failed
→ Salah satu ini:
- DSN tidak sinkron
- Servername ≠ freetds.conf
- FreeTDS belum bisa connect (test tsql!)

## ❌ isql Can't open libtdsodbc.so
→ Salah path driver di odbcinst.ini

## 🏁 Kesimpulan
- PHP 5.6 TIDAK cocok sqlsrv
- FreeTDS + ODBC = solusi paling stabil
- Jika tsql & isql OK → PHP pasti OK
