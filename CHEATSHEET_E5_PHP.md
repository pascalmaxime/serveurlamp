# Cheat Sheet E5 — Partie PHP / Apache / MySQL
> Architecture : Capteurs → M5Stack → MQTT → Backend C++ → MySQL → PHP → Navigateur

---

## 1. LINUX — Commandes rapides utiles

```bash
# Réseau
ip a                          # voir les IPs
ping 192.168.x.x              # tester connectivité
nmcli con show                # connexions réseau

# Services
sudo systemctl start apache2
sudo systemctl start mysql
sudo systemctl start mosquitto
sudo systemctl status apache2
sudo systemctl enable apache2  # démarrage auto

# Firewall UFW
sudo ufw allow 80
sudo ufw allow 1883            # MQTT
sudo ufw allow 3306            # MySQL
sudo ufw enable
sudo ufw status

# Fichiers & droits
sudo chown -R www-data:www-data /var/www/html/
sudo chmod -R 755 /var/www/html/
ls -la /var/www/html/
```

---

## 2. APACHE2 — Virtual Hosts

### Fichier de conf type : `/etc/apache2/sites-available/dashboard.conf`

```apache
<VirtualHost *:80>
    ServerName www.dashboard.local
    DocumentRoot /var/www/dashboard
    <Directory /var/www/dashboard>
        AllowOverride All
        Require all granted
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/dashboard_error.log
</VirtualHost>
```

```bash
# Activer un site
sudo a2ensite dashboard.conf
sudo a2dissite 000-default.conf
sudo systemctl reload apache2

# Activer mod_rewrite (si besoin)
sudo a2enmod rewrite
sudo systemctl restart apache2

# Vérifier conf
sudo apache2ctl configtest

# Logs en direct
sudo tail -f /var/log/apache2/error.log
```

### `/etc/hosts` — résolution locale
```
127.0.0.1   www.nuc.local
127.0.0.1   www.dashboard.local
```

---

## 3. PHP — Bases essentielles

### Connexion MySQL avec PDO (à réutiliser partout)

```php
<?php
$host = 'localhost';
$dbname = 'capteurs_db';
$user = 'root';
$pass = 'motdepasse';

try {
    $pdo = new PDO("mysql:host=$host;dbname=$dbname;charset=utf8", $user, $pass);
    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
} catch (PDOException $e) {
    die("Erreur connexion : " . $e->getMessage());
}
?>
```

### Récupérer des données (SELECT)

```php
<?php
// Tous les relevés
$stmt = $pdo->query("SELECT * FROM releves ORDER BY date_mesure DESC LIMIT 100");
$releves = $stmt->fetchAll(PDO::FETCH_ASSOC);

foreach ($releves as $row) {
    echo $row['temperature'] . " °C<br>";
}

// Avec filtre (requête préparée — TOUJOURS pour les variables)
$capteur = 'bmp280';
$stmt = $pdo->prepare("SELECT * FROM releves WHERE capteur_id = :cid ORDER BY date_mesure DESC");
$stmt->execute([':cid' => $capteur]);
$data = $stmt->fetchAll(PDO::FETCH_ASSOC);
```

### Insérer des données (INSERT)

```php
<?php
$stmt = $pdo->prepare("INSERT INTO releves (capteur_id, temperature, humidite, pression, date_mesure)
                        VALUES (:cid, :temp, :hum, :pres, NOW())");
$stmt->execute([
    ':cid'  => 'bmp280',
    ':temp' => 22.5,
    ':hum'  => 58.3,
    ':pres' => 1013.25
]);
```

### Retourner du JSON (API pour dashboard)

```php
<?php
header('Content-Type: application/json');
header('Access-Control-Allow-Origin: *');  // CORS si besoin

$stmt = $pdo->query("SELECT * FROM releves ORDER BY date_mesure DESC LIMIT 50");
$data = $stmt->fetchAll(PDO::FETCH_ASSOC);

echo json_encode($data);
```

### Lire du JSON en PHP (payload MQTT décodé)

```php
<?php
$payload = '{"temperature": 22.5, "pression": 1013.25, "humidite": 58.3}';
$obj = json_decode($payload, true);  // true = tableau associatif

$temp = $obj['temperature']  ?? -999.0;
$hum  = $obj['humidite']     ?? -999.0;
$pres = $obj['pression']     ?? -999.0;

// Sentinelle : ignorer si tout est absent
if ($temp == -999.0 && $hum == -999.0 && $pres == -999.0) {
    // ne pas insérer
}
```

---

## 4. Dashboard PHP — Template de base

```php
<?php
// Fichier : /var/www/dashboard/index.php
require 'db.php';  // fichier avec connexion PDO

$stmt = $pdo->query("SELECT * FROM releves ORDER BY date_mesure DESC LIMIT 20");
$releves = $stmt->fetchAll(PDO::FETCH_ASSOC);
?>
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="refresh" content="10">  <!-- rafraîchissement auto -->
    <title>Dashboard Capteurs</title>
    <style>
        body { font-family: sans-serif; margin: 20px; background: #f0f0f0; }
        table { border-collapse: collapse; width: 100%; background: white; }
        th, td { border: 1px solid #ccc; padding: 8px 12px; text-align: left; }
        th { background: #007bff; color: white; }
        tr:nth-child(even) { background: #f9f9f9; }
    </style>
</head>
<body>
    <h1>Supervision des capteurs</h1>
    <table>
        <tr>
            <th>Capteur</th>
            <th>Température (°C)</th>
            <th>Humidité (%)</th>
            <th>Pression (hPa)</th>
            <th>Date</th>
        </tr>
        <?php foreach ($releves as $r): ?>
        <tr>
            <td><?= htmlspecialchars($r['capteur_id']) ?></td>
            <td><?= $r['temperature'] != -999 ? $r['temperature'] : '—' ?></td>
            <td><?= $r['humidite']    != -999 ? $r['humidite']    : '—' ?></td>
            <td><?= $r['pression']    != -999 ? $r['pression']    : '—' ?></td>
            <td><?= $r['date_mesure'] ?></td>
        </tr>
        <?php endforeach; ?>
    </table>
</body>
</html>
```

---

## 5. MySQL — Commandes essentielles

```bash
# Connexion
mysql -u root -p

# Ou avec une commande directe
mysql -u root -p capteurs_db -e "SELECT * FROM releves LIMIT 10;"
```

```sql
-- Créer la base et la table
CREATE DATABASE IF NOT EXISTS capteurs_db;
USE capteurs_db;

CREATE TABLE IF NOT EXISTS releves (
    id          INT AUTO_INCREMENT PRIMARY KEY,
    capteur_id  VARCHAR(50)    NOT NULL,
    temperature FLOAT          DEFAULT -999.0,
    humidite    FLOAT          DEFAULT -999.0,
    pression    FLOAT          DEFAULT -999.0,
    co2         FLOAT          DEFAULT -999.0,
    date_mesure DATETIME       DEFAULT NOW()
);

-- Vérifier les données
SELECT * FROM releves ORDER BY date_mesure DESC LIMIT 20;

-- Compter par capteur
SELECT capteur_id, COUNT(*) as nb FROM releves GROUP BY capteur_id;

-- Dernière valeur par capteur
SELECT capteur_id, temperature, humidite, date_mesure
FROM releves
WHERE date_mesure = (SELECT MAX(date_mesure) FROM releves r2 WHERE r2.capteur_id = releves.capteur_id);

-- Créer un utilisateur dédié (si demandé)
CREATE USER 'phpuser'@'localhost' IDENTIFIED BY 'monpassword';
GRANT ALL PRIVILEGES ON capteurs_db.* TO 'phpuser'@'localhost';
FLUSH PRIVILEGES;
```

---

## 6. MQTT — Test rapide (pour aider les collègues)

```bash
# Installer les clients (si pas fait)
sudo apt install mosquitto-clients -y

# S'abonner à tous les topics capteurs
mosquitto_sub -h localhost -t "capteurs/#" -v

# Publier un message de test
mosquitto_pub -h localhost -t "capteurs/bmp280" \
  -m '{"temperature": 22.5, "pression": 1013.25, "humidite": 58.3}'

# Avec authentification (si Mosquitto sécurisé)
mosquitto_sub -h localhost -u utilisateur -P motdepasse -t "capteurs/#" -v
mosquitto_pub -h localhost -u utilisateur -P motdepasse \
  -t "capteurs/bmp280" -m '{"temperature": 22.5, "pression": 1013.25, "humidite": 58.3}'

# Config Mosquitto : /etc/mosquitto/mosquitto.conf
# Vérifier les logs
sudo journalctl -u mosquitto -f
```

---

## 7. PHP — Fonctions utiles à connaître

```php
// Dates
date('Y-m-d H:i:s')            // format MySQL DATETIME
date('d/m/Y H:i')              // format affichage

// Sécurité affichage (toujours)
htmlspecialchars($str)          // évite XSS

// Vérifications
isset($variable)                // variable définie ?
empty($variable)                // vide ou non défini ?
is_numeric($val)                // c'est un nombre ?

// Superglobales
$_GET['param']                  // depuis URL ?param=valeur
$_POST['champ']                 // depuis formulaire POST
$_SERVER['REQUEST_METHOD']      // GET ou POST

// Redirection
header('Location: index.php');
exit;

// Inclure un fichier
require 'config.php';           // erreur fatale si absent
include 'header.php';           // warning si absent
```

---

## 8. Structure de fichiers suggérée

```
/var/www/dashboard/
├── index.php          ← tableau de bord principal
├── db.php             ← connexion PDO (require dans les autres)
├── api.php            ← endpoint JSON (optionnel)
└── style.css          ← feuille de style (optionnel)

/var/www/html/         ← site statique (www.nuc.local)
└── index.html
```

---

## 9. Checklist rapide pour le TP

- [ ] Apache2 lancé et virtual hosts configurés
- [ ] `/etc/hosts` mis à jour sur la machine
- [ ] MySQL lancé, base `capteurs_db` créée, table `releves` créée
- [ ] Connexion PDO testée (afficher un `var_dump($pdo)`)
- [ ] Le backend C++ insère bien des données → vérifier avec `SELECT`
- [ ] Le dashboard affiche les données en tableau
- [ ] Rafraîchissement auto (`<meta http-equiv="refresh" content="10">`)
- [ ] Valeurs sentinelles `-999` affichées comme `—`

---

*Référence perso — épreuve E5 BTS*
