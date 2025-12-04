# Lab10Web
#### Nama   = DZAKI ARIF RAHMAN  
#### Kelas  = TI.24.A4  
#### NIM    = 312410312  
#### Matkul  = Pemograman Web 1 



## struktur nya
<img width="134" height="262" alt="image" src="https://github.com/user-attachments/assets/b54e7090-c318-48d8-afc9-fb7d9cc27fa0" />

Saya lampirkan hasil pengerjaan lanjutan tugas Praktikum 10, yaitu Modularisasi Aplikasi CRUD Data Barang menggunakan konsep Object-Oriented Programming (OOP).

Tujuan utama dari tugas ini adalah untuk memisahkan fungsi-fungsi krusial (Koneksi Database dan Form Input) dari kode utama,

## 1\. Class Library (Komponen Modular)

Saya telah membuat tiga file utama yang berfungsi sebagai modul.

### 1.1 `config.php` (Konfigurasi Database)

**Penjelasan:** Saya membuat file ini agar detail koneksi database tidak tersebar di banyak tempat. File ini menyimpan *setting* server, *username*, *password*, dan nama database **`latihan1`** dalam sebuah *array*. *Class* `Database` akan menggunakan *array* ini.

```php
<?php
$config = array(
    'host'     => 'localhost',
    'username' => 'root',     
    'password' => '',         
    'db_name'  => 'latihan1'  
);
```

### 1.2 `database.php` (Class Library Database)

**Penjelasan:** Ini adalah **Class `Database`** yang saya buat untuk menggantikan file `koneksi.php` yang lama. Semua fungsi koneksi dan query prosedural (`mysqli_*`) sudah saya pindahkan ke sini menjadi *method* OOP. Koneksi dibuat secara otomatis di *method* `__construct()`. Sekarang, untuk menjalankan query, saya hanya perlu memanggil *method* seperti `insert()`, `getAll()`, `update()`, atau `delete()`, membuat kode CRUD saya jauh lebih bersih.

```php
<?php
class Database {
    protected $host;
    protected $user;
    protected $password;
    protected $db_name;
    protected $conn;

    public function __construct() { 
        $this->getConfig();
        $this->conn = new mysqli($this->host, $this->user, $this->password, $this->db_name); 
        
        if ($this->conn->connect_error) {
            die("Connection failed: " . $this->conn->connect_error);
        }
    }
    
    private function getConfig() {
        include_once("config.php");
        $this->host = $config['host'];
        $this->user = $config['username'];
        $this->password = $config['password'];
        $this->db_name = $config['db_name'];
    }
    
    public function getAll($table, $order_by=null) {
        $order_clause = $order_by ? " ORDER BY " . $order_by : "";
        $sql = "SELECT * FROM " . $table . $order_clause;
        return $this->conn->query($sql);
    }

    public function insert($table, $data) {
        if (is_array($data)) {
            $column = [];
            $value = [];
            foreach($data as $key => $val) {
                $val = $this->conn->real_escape_string($val); 
                $column[] = $key;
                $value[] = "'{$val}'";
            }
            $columns = implode(",", $column);
            $values  = implode(",", $value);
        }
        $sql = "INSERT INTO " . $table . " (" . $columns . ") VALUES (" . $values . ")";
        $result = $this->conn->query($sql);
        return $result;
    }
    
    public function getOne($table, $where) {
        $sql = "SELECT * FROM " . $table . " WHERE " . $where;
        $result = $this->conn->query($sql); 
        return $result ? $result->fetch_assoc() : null;
    }
    
    public function update($table, $data, $where) {
        $updates = [];
        if (is_array($data)) {
            foreach($data as $key => $val) {
                $val = $this->conn->real_escape_string($val);
                $updates[] = "$key='{$val}'";
            }
            $update_value = implode(",", $updates);
        }
        $sql = "UPDATE " . $table . " SET " . $update_value . " WHERE " . $where;
        $result = $this->conn->query($sql);
        return $result;
    }

    public function delete($table, $filter) {
        $sql = "DELETE FROM " . $table . " WHERE " . $filter;
        $result = $this->conn->query($sql);
        return $result;
    }
}
```

### 1.3 `form.php` (Class Library Form)

**Penjelasan:** Saya membuat **Class `Form`** untuk memodularisasi pembuatan elemen formulir. Saya menambahkan *method* **`addSelect()`** sendiri untuk mendukung *dropdown* Kategori. Logika yang sebelumnya error saat mengakses *array* `'value'` juga sudah saya perbaiki di `displayForm()` dengan menambahkan `?? ''`.

```php
<?php
class Form
{
    private $fields = array();
    private $action;
    private $submit = "Submit Form";
    private $jumField = 0;

    public function __construct($action, $submit)
    {
        $this->action = $action;
        $this->submit = $submit;
    }

    public function addField($name, $label, $type="text", $value="", $required=true)
    {
        $this->fields [$this->jumField] = [
            'type' => $type,
            'name' => $name,
            'label' => $label,
            'value' => $value,
            'required' => $required
        ];
        $this->jumField ++;
    }

    public function addSelect($name, $label, $options, $selectedValue="", $required=true)
    {
        $this->fields [$this->jumField] = [
            'type' => 'select',
            'name' => $name,
            'label' => $label,
            'options' => $options,
            'selected' => $selectedValue,
            'required' => $required
        ];
        $this->jumField ++;
    }

    public function displayForm()
    {
        echo "<form action='". $this->action . "' method='POST' enctype='multipart/form-data'>";
        echo '<div class="main">';
        
        for ($j=0; $j<count($this->fields); $j++) {
            $field = $this->fields[$j];
            
            if ($field['type'] == 'hidden') {
                echo "<input type='hidden' name='{$field['name']}' value='". htmlspecialchars($field['value']) . "' />";
                continue;
            }

            $required = $field['required'] ? 'required' : '';
            // FIX: Mengatasi Undefined Array Key 'value'
            $value = htmlspecialchars($field['value'] ?? ''); 
            
            echo '<div class="input">';
            echo '<label>' . htmlspecialchars($field['label']) . '</label>';

            if ($field['type'] == 'select') {
                echo "<select name='{$field['name']}' {$required}>";
                echo "<option value=''>-- Pilih {$field['label']} --</option>";
                foreach ($field['options'] as $opt) {
                    $selected = ($field['selected'] == $opt) ? 'selected' : '';
                    echo "<option value='{$opt}' {$selected}>{$opt}</option>";
                }
                echo "</select>";
            } else {
                echo "<input type='{$field['type']}' name='{$field['name']}' value='{$value}' {$required}/>";
            }
            echo '</div>';
        }
        
        echo "<div class='submit'>";
        echo "<input type='submit' name='submit' value='". $this->submit . "' />";
        echo "</div>";
        echo "</div>";
        echo "</form>"; 
    }
}
```

-----

## 2\. File Aplikasi CRUD (Penggunaan Modular)

### 2.1 `index.php`

**Penjelasan:** Saya mengganti `include('koneksi.php')` dengan `include_once("database.php")`. Untuk menampilkan data, saya cukup memanggil `$db->getAll()` dan memproses hasilnya.

```php
<?php
include_once("database.php");

$db = new Database();

// Mengambil data menggunakan method Class Database
$result  = $db->getAll('data_barang', 'id_barang DESC');
?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <link href="style.css" rel="stylesheet" type="text/css" />
    <title>Data Barang</title>
</head>
<body>
    <div class="container">
        <h1>Data Barang</h1>
        <a href="tambah.php" class="btn-tambah">Tambah Barang</a> 
        <div class="main">
            <table>
                <tr>
                    <th>Gambar</th>
                    <th>Nama Barang</th>
                    <th>Kategori</th>
                    <th>Harga Jual</th>
                    <th>Harga Beli</th>
                    <th>Stok</th>
                    <th>Aksi</th>
                </tr>
                <?php if($result && $result->num_rows > 0): ?> 
                <?php while($row = $result->fetch_array()): ?> 
                <tr>
                    <td>
                        <?php if ($row['gambar']): ?>
                            <img src="gambar/<?= htmlspecialchars($row['gambar']);?>" alt="<?= htmlspecialchars($row['nama']);?>" style="max-width: 100px; max-height: 100px;">
                        <?php else: ?>
                            Tidak Ada Gambar
                        <?php endif; ?>
                    </td> 
                    
                    <td><?= htmlspecialchars($row['nama']);?></td>
                    <td><?= htmlspecialchars($row['kategori']);?></td>
                    <td><?= htmlspecialchars($row['harga_jual']);?></td>
                    <td><?= htmlspecialchars($row['harga_beli']);?></td>
                    <td><?= htmlspecialchars($row['stok']);?></td>
                    <td>
                        <a href="ubah.php?id=<?= $row['id_barang'];?>">Ubah</a> 
                        <a href="hapus.php?id=<?= $row['id_barang'];?>" onclick="return confirm('Yakin akan menghapus data ini?')">Hapus</a>
                    </td>
                </tr>
                <?php endwhile; else: ?>
                <tr>
                    <td colspan="7">Belum ada data di database.</td>
                </tr>
                <?php endif; ?>
            </table>
        </div>
    </div>
</body>
</html>
```

### 3.2 `tambah.php`

**Penjelasan:** Logika INSERT datanya menggunakan `$db->insert()`. Kemudian, form-nya dibuat sepenuhnya oleh objek `$form` menggunakan *method* `addField()` dan `addSelect()`.

```php
<?php
error_reporting(E_ALL);
include_once("database.php");
include_once("form.php"); 

$db = new Database(); 

if (isset($_POST['submit']))
{
    $nama       = $_POST['nama'];
    $kategori   = $_POST['kategori'];
    $harga_jual = $_POST['harga_jual'];
    $harga_beli = $_POST['harga_beli'];
    $stok       = $_POST['stok'];
    $file_gambar = $_FILES['file_gambar'];
    $gambar     = null;

    // Proses upload gambar
    if ($file_gambar ['error'] == 0) 
    {
        $filename    = str_replace(' ', '_', $file_gambar ['name']);
        $destination = dirname(__FILE__) . '/gambar/'. $filename; 
        if(move_uploaded_file($file_gambar ['tmp_name'], $destination)) 
        {
            $gambar = $filename; 
        }
    }
    
    $data_insert = [
        'nama' => $nama, 
        'kategori' => $kategori, 
        'harga_jual' => $harga_jual, 
        'harga_beli' => $harga_beli, 
        'stok' => $stok, 
        'gambar' => $gambar 
    ];

    // Menggunakan method insert() Class Database
    $result = $db->insert('data_barang', $data_insert);

    if ($result) {
        header('location: index.php');
        exit;
    } else {
        $error = "Gagal menyimpan data ke database.";
    }
}
?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <link href="style.css" rel="stylesheet" type="text/css" />
    <title>Tambah Barang</title>
</head>
<body>
    <div class="container">
        <h1>Tambah Barang</h1>
        <div class="main">
            <?php 
                if (isset($error)) {
                    echo "<p style='color: red;'>$error</p>";
                } 
                
                $form = new Form("tambah.php", "Simpan"); 
                
                // Fields dibuat secara modular
                $form->addField("nama", "Nama Barang");
                $form->addSelect("kategori", "Kategori", ['Komputer', 'Elektronik', 'Hand Phone']);
                $form->addField("harga_jual", "Harga Jual", "number");
                $form->addField("harga_beli", "Harga Beli", "number");
                $form->addField("stok", "Stok", "number");
                $form->addField("file_gambar", "File Gambar (Kosongkan jika tidak diubah)", "file", "", false);
                
                $form->displayForm(); 
            ?>
        </div>
    </div>
</body>
</html>
```

### 3.3 `ubah.php`

**Penjelasan:** Logika pengambilan data lama menggunakan `$db->getOne()`. Logika *update* menggunakan `$db->update()`. Form-nya dibuat menggunakan *Class* `Form`, dan nilai lama langsung dimasukkan sebagai *value* di *method* `addField()` dan `addSelect()`.

```php
<?php
error_reporting (E_ALL);
include_once("database.php");
include_once("form.php"); 

$db = new Database(); 

// 1. Logika Pemrosesan Form UPDATE saat di-submit
if (isset($_POST['submit']))
{
    $id = $_POST['id'];
    $nama = $_POST['nama'];
    $kategori = $_POST['kategori'];
    $harga_jual = $_POST['harga_jual'];
    $harga_beli = $_POST['harga_beli'];
    $stok = $_POST['stok'];
    $file_gambar = $_FILES['file_gambar'];
    $gambar_lama = $_POST['gambar_lama'];
    $gambar = $gambar_lama;

    // Proses upload gambar baru (jika ada)
    if ($file_gambar ['error'] == 0)
    {
        $filename = str_replace(' ', '_', $file_gambar['name']);
        $destination = dirname(__FILE__) . '/gambar/' . $filename;
        
        if(move_uploaded_file($file_gambar ['tmp_name'], $destination)) {
            $gambar = $filename;
            // Hapus gambar lama jika ada
            if ($gambar_lama && file_exists(dirname(__FILE__) . '/gambar/' . $gambar_lama)) {
                unlink(dirname(__FILE__) . '/gambar/' . $gambar_lama);
            }
        }
    }

    $data_update = [
        'nama' => $nama, 
        'kategori' => $kategori, 
        'harga_jual' => $harga_jual, 
        'harga_beli' => $harga_beli, 
        'stok' => $stok, 
        'gambar' => $gambar 
    ];

    // Query UPDATE menggunakan method update() Class Database
    $result = $db->update('data_barang', $data_update, "id_barang='{$id}'");

    if ($result) {
        header('location: index.php');
        exit;
    } else {
        $error = "Gagal mengubah data.";
    }
} 

// 2. Logika Pengambilan Data Barang
if (isset($_GET['id'])) {
    $id = $_GET['id'];
    // Menggunakan method getOne() dari Class Database
    $data = $db->getOne('data_barang', "id_barang='{$id}'");

    if (!$data) {
        header('location: index.php');
        exit;
    }
} else {
    header('location: index.php');
    exit;
}
?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <link href="style.css" rel="stylesheet" type="text/css" />
    <title>Ubah Barang</title>
</head>
<body>
    <div class="container">
        <h1>Ubah Barang</h1>
        <div class="main">
            <?php 
                if (isset($error)) {
                    echo "<p style='color: red;'>$error</p>";
                }
                
                $form = new Form("ubah.php", "Simpan Perubahan"); 
                
                // Fields diisi dengan data lama
                $form->addField("id", "", "hidden", $data['id_barang'], false); 
                $form->addField("gambar_lama", "", "hidden", $data['gambar'], false); 

                $form->addField("nama", "Nama Barang", "text", $data['nama']);
                $form->addSelect("kategori", "Kategori", ['Komputer', 'Elektronik', 'Hand Phone'], $data['kategori']);
                $form->addField("harga_jual", "Harga Jual", "number", $data['harga_jual']);
                $form->addField("harga_beli", "Harga Beli", "number", $data['harga_beli']);
                $form->addField("stok", "Stok", "number", $data['stok']);
                $form->addField("file_gambar", "File Gambar (Kosongkan jika tidak diubah)", "file", "", false);
                
                $form->displayForm();
            ?>
            <?php if ($data['gambar']): ?>
                <p>Gambar Saat Ini: 
                    <img src="gambar/<?php echo htmlspecialchars($data['gambar']);?>" style="max-width: 100px; max-height: 100px; display: block; margin-top: 10px;">
                </p>
            <?php endif; ?>
        </div>
    </div>
</body>
</html>
```

### 3.4 `hapus.php`

**Penjelasan:** File ini sekarang sangat sederhana, hanya menggunakan `$db->delete()` untuk menjalankan query hapus.

```php
<?php
include_once 'database.php';

$db = new Database(); 

if (isset($_GET['id'])) {
    $id = $_GET['id'];
    
    // Ambil nama file gambar lama sebelum dihapus
    $data = $db->getOne('data_barang', "id_barang='{$id}'");
    $gambar_lama = $data['gambar'] ?? null;

    // Query DELETE menggunakan method delete() Class Database
    $result = $db->delete('data_barang', "id_barang = '{$id}'");

    if ($result) {
        // Hapus file gambar dari server jika ada
        if ($gambar_lama && file_exists(dirname(__FILE__) . '/gambar/' . $gambar_lama)) {
            unlink(dirname(__FILE__) . '/gambar/' . $gambar_lama);
        }
    } else {
        die("Gagal menghapus data.");
    }
}

header('location: index.php');
exit;
?>
```
## 4\. Kode Tambahan (Demonstrasi OOP Dasar)

Saya juga menyimpan file ini terpisah sebagai bukti pemahaman saya tentang konsep dasar *Class* dan *Object* di PHP.

### 4.1 `mobil.php`

```php
<?php
class Mobil
{
    private $warna;
    private $merk;
    private $harga;

    public function __construct() 
    {
        $this->warna = "Biru";
        $this->merk = "BMW";
        $this->harga = "10000000";
    }

    public function gantiwarna ($warnaBaru)
    {
        $this->warna = $warnaBaru; 
    }

    public function tampilWarna ()
    {
        echo "Warna mobilnya: ";
        echo $this->warna;
    }
}

$a = new Mobil();
$b = new Mobil();

echo "<b>Mobil pertama</b><br>";
$a->tampilWarna(); 
echo "<br>Mobil pertama ganti warna<br>";
$a->gantiwarna ("Merah");
$a->tampilWarna(); 

echo "<br><b>Mobil kedua</b><br>";
$b->gantiwarna("Hijau");
$b->tampilWarna(); 
```
screenshot

<img width="1537" height="808" alt="Screenshot 2025-12-04 134216" src="https://github.com/user-attachments/assets/ea5f3bb1-bdc0-453b-bc37-d6d2904ad610" />


-----

## 5\. screenshot kodingannya

**Tampilan awal**
<img width="1534" height="933" alt="image" src="https://github.com/user-attachments/assets/60e79b2c-a97c-494e-a461-5f50027a4ebc" />


**Tampilan Tambahkan Prodak**
<img width="1230" height="827" alt="image" src="https://github.com/user-attachments/assets/b797aad7-b4ca-4c6b-9e1c-8acbf40ca769" />





