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

### 6.2 Compile pdo_odbc dan Build pakai php-config aaPanel
```
cd /usr/local/src
wget https://www.php.net/distributions/php-5.6.40.tar.gz
tar zxvf php-5.6.40.tar.gz
cd php-5.6.40/ext/pdo_odbc
/www/server/php/56/bin/phpize

./configure \
 --with-php-config=/www/server/php/56/bin/php-config \
 --with-pdo-odbc=unixODBC,/usr

make
make install
```

### 6.3 compile odbc dan Build pakai php-config aaPanel (ODBC INI MASIH GAGAL DIPAKAI)
```
# Buat folder yang dicari configure
sudo mkdir -p /usr/local/incl
sudo ln -s /usr/include/sqlext.h /usr/local/incl/sqlext.h
sudo ln -s /usr/include/sql.h /usr/local/incl/sql.h
sudo ln -s /usr/include/sqltypes.h /usr/local/incl/sqltypes.h

# Pastikan library ODBC ada
sudo mkdir -p /usr/local/incl
sudo ln -s /usr/include/sqlext.h /usr/local/incl/sqlext.h
sudo ln -s /usr/include/sql.h /usr/local/incl/sql.h
sudo ln -s /usr/include/sqltypes.h /usr/local/incl/sqltypes.h

# Compile ODBC PHP extension
cd php-5.6.40/ext/odbc
/www/server/php/56/bin/phpize
./configure \
  --with-php-config=/www/server/php/56/bin/php-config \
  --with-unixODBC=shared,/usr
make
make install
```

### 6.4 Enable extension

📍 `/www/server/php/56/etc/php.ini`
```
extension=pdo.so
extension=pdo_odbc.so
extension=odbc.so
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

-------------------------------------------------

Install & Aktifkan MSSQL (Native) di PHP 5.6 Linux menggunakan FreeTDS

Target:
PHP 5.6 + Linux + MSSQL Server (SQL Server 2017)
Driver: mssql.so (native, non-PDO)
Cocok untuk ADODB + PHPMaker (legacy)

1️⃣ Install FreeTDS (Library MSSQL)
apt update
apt install -y freetds-dev freetds-bin unixodbc unixodbc-dev


Pastikan library ada:

ls /usr/lib/x86_64-linux-gnu/libsybdb.so*

2️⃣ Buat symlink agar kompatibel dengan PHP 5.6 (WAJIB)

PHP 5.6 ext/mssql hanya mencari /usr/lib.

ln -s /usr/lib/x86_64-linux-gnu/libsybdb.so /usr/lib/libsybdb.so
ln -s /usr/lib/x86_64-linux-gnu/libsybdb.so /usr/lib/libsybdb.a


Refresh linker:

ldconfig


Cek:

ldconfig -p | grep sybdb

3️⃣ Konfigurasi FreeTDS

Edit:

vi /etc/freetds/freetds.conf


Tambahkan:

[sql2017_samator]
    host = 202.155.143.234
    port = 1433
    tds version = 7.4

4️⃣ Test koneksi FreeTDS (PENTING)
tsql -S sql2017_samator -U saw -P 123456


Jika masuk prompt:

1>


✅ FreeTDS sudah OK

5️⃣ Compile ekstensi mssql untuk PHP 5.6
cd /tmp
wget https://www.php.net/distributions/php-5.6.30.tar.gz
tar zxvf php-5.6.30.tar.gz

cd php-5.6.30/ext/mssql

/www/server/php/56/bin/phpize

./configure \
  --with-php-config=/www/server/php/56/bin/php-config \
  --with-mssql=/usr

make
make install

6️⃣ Aktifkan extension

Edit:

vi /www/server/php/56/etc/php.ini


Tambahkan:

extension=mssql.so


Restart:

service php-fpm-56 restart
service nginx restart

7️⃣ Verifikasi PHP
/www/server/php/56/bin/php -m | grep mssql


Output:

mssql

8️⃣ Test PHP Native MSSQL
<?php
$conn = mssql_connect("sql2017_samator", "saw", "123456");
if (!$conn) {
    die(mssql_get_last_message());
}

mssql_select_db("Surabaya_System", $conn);

$q = mssql_query("SELECT TOP 5 name FROM sys.tables");
while ($r = mssql_fetch_assoc($q)) {
    echo $r['name']."<br>";
}


✅ Jika data tampil → SUKSES TOTAL

9️⃣ Gunakan di ADODB / PHPMaker
$conn = ADONewConnection("mssql");
$conn->Connect("sql2017_samator", "saw", "123456", "Surabaya_System");


Tidak perlu:

❌ PDO_ODBC

❌ sqlsrv

❌ pdo_sqlsrv

🧠 CATATAN PENTING

PHP 5.6 TIDAK stabil dengan PDO_ODBC

mssql.so + FreeTDS adalah solusi legacy paling stabil

Cocok untuk:

ADODB lama

PHPMaker 2019–2020

Aplikasi warisan (legacy system)
