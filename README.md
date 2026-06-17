# SIS313 — Proyecto Final | Grupo 6 | VLAN 206
## Despliegue de Microservicios con Automatización

**Universidad San Francisco Xavier de Chuquisaca**  
**Carrera:** Ingeniería de Sistemas  
**Asignatura:** Infraestructura, Plataformas Tecnológicas y Redes — SIS313  
**Docente:** Ing. Marcelo Quispe Ortega  
**Semestre:** 1/2026  

---

## Integrantes

| Nombre | Servidores | Rol |
|---|---|---|
| Marco Antonio Sanchez Galarza | server-159, server-160, server-161 | MariaDB Master + Slave + Prometheus + Grafana |
| Jose Luis Miranda Ruiz | server-157, server-158 | NGINX Proxy + Apps Node.js + PM2 |

---

## Arquitectura del Sistema

```
Internet
    │
    ▼
[Jump/Proxy] 201.131.45.42
    │  NGINX — balanceo least_conn
    ├──────────────────────┐
    ▼                      ▼
[server-157]          [server-158]
App 1 Node.js         App 2 Node.js
Puerto 3001           Puerto 3001
    │                      │
    └──────────┬───────────┘
               ▼
         [server-159]
       MariaDB Master
       192.168.100.159
               │
               │ replicación
               ▼
         [server-160]
        MariaDB Slave
       192.168.100.160

         [server-161]
    Prometheus + Grafana
       192.168.100.161
    (monitorea los 5 servidores)
```

| Servidor | IP | Rol |
|---|---|---|
| Jump/Proxy | 201.131.45.42 | NGINX Balanceador (docente) |
| server-157 | 192.168.100.157 | App 1 — Node.js + PM2 |
| server-158 | 192.168.100.158 | App 2 — Node.js + PM2 |
| server-159 | 192.168.100.159 | MariaDB Master |
| server-160 | 192.168.100.160 | MariaDB Slave |
| server-161 | 192.168.100.161 | Prometheus + Grafana |

---

## Tecnologías Utilizadas

- **Node.js 18 + PM2** — microservicios y gestión de procesos
- **NGINX** — proxy inverso con balanceo `least_conn`
- **MariaDB** — base de datos con replicación Master-Slave
- **Prometheus + Grafana** — monitoreo y visualización de métricas
- **UFW** — firewall con política de mínimo privilegio
- **Bash** — scripts de automatización, deploy, health check y backup

---

## Microservicios

Cada servidor de aplicación (157 y 158) expone los mismos 3 endpoints:

| Endpoint | Método | Descripción |
|---|---|---|
| `/usuarios` | GET / POST | Gestión de usuarios |
| `/pagos` | GET / POST | Gestión de pagos |
| `/notificaciones` | GET | Notificaciones del sistema |
| `/health` | GET | Estado del servicio |

---

## Server-157 — App 1 (Node.js + PM2)

### Conexión
```bash
# Primero entrar al jump server
ssh usrproxy@201.131.45.42

# Luego conectarse a server-157
ssh adming6@192.168.100.157
```

### Instalación de Node.js 18
```bash
# Actualizar repositorios
sudo apt update

# Agregar repositorio oficial de NodeSource para Node.js 18
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -

# Instalar Node.js
sudo apt install nodejs -y

# Verificar versiones instaladas
node --version   # v18.x.x
npm --version    # 10.x.x
```

### Instalación de PM2
```bash
# Instalar PM2 globalmente — gestor de procesos para Node.js en producción
# Permite que la app se reinicie automáticamente si falla
sudo npm install -g pm2

# Verificar instalación
pm2 --version
```

### Crear la aplicación
```bash
# Crear directorio del proyecto
mkdir -p ~/app && cd ~/app

# Inicializar proyecto Node.js
npm init -y

# Instalar dependencias:
# express — framework web
# mysql2  — cliente MariaDB con soporte de promesas
# dotenv  — carga variables de entorno desde .env
npm install express mysql2 dotenv
```

### Archivo de variables de entorno
```bash
nano ~/app/.env
```

```env
PORT=3001
APP_NAME=app1-server157
DB_HOST=192.168.100.159
DB_PORT=3306
DB_USER=appuser
DB_PASS=App1234!
DB_NAME=proyecto_db
```

> `DB_HOST` apunta al server-159 (MariaDB Master). Las credenciales corresponden al usuario `appuser` creado en la BD con permisos sobre `proyecto_db`.

### Código principal — index.js
```bash
nano ~/app/index.js
```

```javascript
const express = require('express');
const mysql = require('mysql2/promise');
require('dotenv').config();

const app = express();
const PORT = process.env.PORT || 3001;
app.use(express.json());

// Pool de conexiones a MariaDB — reutiliza conexiones en lugar de abrir una nueva por cada request
const pool = mysql.createPool({
  host:     process.env.DB_HOST,
  port:     process.env.DB_PORT,
  user:     process.env.DB_USER,
  password: process.env.DB_PASS,
  database: process.env.DB_NAME,
  waitForConnections: true,
  connectionLimit: 10,
});

// --- MICROSERVICIO USUARIOS ---
app.get('/usuarios', async (req, res) => {
  try {
    const [rows] = await pool.query('SELECT * FROM usuarios');
    res.json({ ok: true, servicio: 'usuarios', vm: process.env.APP_NAME, data: rows });
  } catch(err) { res.status(500).json({ ok: false, error: err.message }); }
});

app.post('/usuarios', async (req, res) => {
  try {
    const { nombre, email } = req.body;
    const [r] = await pool.query('INSERT INTO usuarios (nombre,email) VALUES (?,?)', [nombre, email]);
    res.status(201).json({ ok: true, id: r.insertId });
  } catch(err) { res.status(500).json({ ok: false, error: err.message }); }
});

// --- MICROSERVICIO PAGOS ---
app.get('/pagos', async (req, res) => {
  try {
    const [rows] = await pool.query('SELECT * FROM pagos');
    res.json({ ok: true, servicio: 'pagos', vm: process.env.APP_NAME, data: rows });
  } catch(err) { res.status(500).json({ ok: false, error: err.message }); }
});

app.post('/pagos', async (req, res) => {
  try {
    const { usuario_id, monto, concepto } = req.body;
    const [r] = await pool.query('INSERT INTO pagos (usuario_id,monto,concepto) VALUES (?,?,?)', [usuario_id, monto, concepto]);
    res.status(201).json({ ok: true, id: r.insertId });
  } catch(err) { res.status(500).json({ ok: false, error: err.message }); }
});

// --- MICROSERVICIO NOTIFICACIONES ---
app.get('/notificaciones', async (req, res) => {
  try {
    const [rows] = await pool.query('SELECT * FROM notificaciones');
    res.json({ ok: true, servicio: 'notificaciones', vm: process.env.APP_NAME, data: rows });
  } catch(err) { res.status(500).json({ ok: false, error: err.message }); }
});

// --- HEALTH Y ROOT ---
app.get('/', (req, res) => {
  res.json({ servicio: 'microservicios', vm: process.env.APP_NAME, estado: 'operativo' });
});

app.get('/health', (req, res) => {
  res.json({ status: 'ok', vm: process.env.APP_NAME, puerto: PORT });
});

app.listen(PORT, '0.0.0.0', () => {
  console.log('App corriendo en puerto ' + PORT + ' — ' + process.env.APP_NAME);
});
```

### Lanzar con PM2
```bash
cd ~/app

# Iniciar la app con PM2 y asignarle un nombre identificable
pm2 start index.js --name app1-server157

# Verificar que está corriendo
pm2 list
```

Resultado esperado:
```
┌────┬──────────────────┬────────┬───────────┐
│ id │ name             │ mode   │ status    │
├────┼──────────────────┼────────┼───────────┤
│  0 │ app1-server157   │ fork   │ online    │
└────┴──────────────────┴────────┴───────────┘
```

```bash
# Configurar PM2 para que inicie automáticamente al reiniciar el servidor
pm2 startup
# Copiar y ejecutar el comando que PM2 muestra en pantalla

# Guardar la lista de procesos actual
pm2 save
```

### Verificar endpoints
```bash
curl http://localhost:3001/
curl http://localhost:3001/health
curl http://localhost:3001/usuarios
curl http://localhost:3001/pagos
curl http://localhost:3001/notificaciones
```

Resultado esperado en `/usuarios`:
```json
{"ok":true,"servicio":"usuarios","vm":"app1-server157","data":[...]}
```

### Firewall UFW
```bash
# Denegar todo el tráfico entrante por defecto (principio de mínimo privilegio)
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Permitir solo los puertos necesarios
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 3001/tcp  # App Node.js
sudo ufw allow 9100/tcp  # Node Exporter (métricas para Prometheus)

# Activar firewall
sudo ufw --force enable
sudo ufw status verbose
```

### Hardening SSH
```bash
# Deshabilitar login como root — elimina el usuario más atacado
sudo sed -i 's/^#*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config

# Limitar intentos de autenticación fallidos a 3
sudo sed -i 's/^#*MaxAuthTries.*/MaxAuthTries 3/' /etc/ssh/sshd_config

sudo systemctl restart ssh

# Verificar que los cambios fueron aplicados
grep -E "^(PermitRootLogin|MaxAuthTries)" /etc/ssh/sshd_config
```

Resultado esperado:
```
PermitRootLogin no
MaxAuthTries 3
```

### Node Exporter (métricas para Prometheus)
```bash
# Instalar agente que expone métricas del SO en el puerto 9100
sudo apt install prometheus-node-exporter -y
sudo systemctl enable prometheus-node-exporter
sudo systemctl start prometheus-node-exporter

# Verificar que las métricas están disponibles
curl http://localhost:9100/metrics | head -5
```


---

## Server-158 — App 2 (Node.js + PM2)

### Conexión
```bash
# Primero entrar al jump server
ssh usrproxy@201.131.45.42

# Luego conectarse a server-158
ssh adming6@192.168.100.158
```

### Instalación de Node.js 18
```bash
# Actualizar repositorios
sudo apt update

# Agregar repositorio oficial de NodeSource para Node.js 18
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -

# Instalar Node.js
sudo apt install nodejs -y

# Verificar versiones instaladas
node --version   # v18.x.x
npm --version    # 10.x.x
```

### Instalación de PM2
```bash
# Instalar PM2 globalmente — gestor de procesos para Node.js en producción
# Permite que la app se reinicie automáticamente si falla
sudo npm install -g pm2

# Verificar instalación
pm2 --version
```

### Crear la aplicación
```bash
# Crear directorio del proyecto
mkdir -p ~/app && cd ~/app

# Inicializar proyecto Node.js
npm init -y

# Instalar dependencias:
# express — framework web
# mysql2  — cliente MariaDB con soporte de promesas
# dotenv  — carga variables de entorno desde .env
npm install express mysql2 dotenv
```

### Archivo de variables de entorno
```bash
nano ~/app/.env
```

```env
PORT=3001
APP_NAME=app2-server158
DB_HOST=192.168.100.159
DB_PORT=3306
DB_USER=appuser
DB_PASS=App1234!
DB_NAME=proyecto_db
```

> `DB_HOST` apunta al server-159 (MariaDB Master). Las credenciales corresponden al usuario `appuser` creado en la BD con permisos sobre `proyecto_db`.

### Código principal — index.js
```bash
nano ~/app/index.js
```

```javascript
const express = require('express');
const mysql = require('mysql2/promise');
require('dotenv').config();

const app = express();
const PORT = process.env.PORT || 3001;
app.use(express.json());

// Pool de conexiones a MariaDB — reutiliza conexiones en lugar de abrir una nueva por cada request
const pool = mysql.createPool({
  host:     process.env.DB_HOST,
  port:     process.env.DB_PORT,
  user:     process.env.DB_USER,
  password: process.env.DB_PASS,
  database: process.env.DB_NAME,
  waitForConnections: true,
  connectionLimit: 10,
});

// --- MICROSERVICIO USUARIOS ---
app.get('/usuarios', async (req, res) => {
  try {
    const [rows] = await pool.query('SELECT * FROM usuarios');
    res.json({ ok: true, servicio: 'usuarios', vm: process.env.APP_NAME, data: rows });
  } catch(err) { res.status(500).json({ ok: false, error: err.message }); }
});

app.post('/usuarios', async (req, res) => {
  try {
    const { nombre, email } = req.body;
    const [r] = await pool.query('INSERT INTO usuarios (nombre,email) VALUES (?,?)', [nombre, email]);
    res.status(201).json({ ok: true, id: r.insertId });
  } catch(err) { res.status(500).json({ ok: false, error: err.message }); }
});

// --- MICROSERVICIO PAGOS ---
app.get('/pagos', async (req, res) => {
  try {
    const [rows] = await pool.query('SELECT * FROM pagos');
    res.json({ ok: true, servicio: 'pagos', vm: process.env.APP_NAME, data: rows });
  } catch(err) { res.status(500).json({ ok: false, error: err.message }); }
});

app.post('/pagos', async (req, res) => {
  try {
    const { usuario_id, monto, concepto } = req.body;
    const [r] = await pool.query('INSERT INTO pagos (usuario_id,monto,concepto) VALUES (?,?,?)', [usuario_id, monto, concepto]);
    res.status(201).json({ ok: true, id: r.insertId });
  } catch(err) { res.status(500).json({ ok: false, error: err.message }); }
});

// --- MICROSERVICIO NOTIFICACIONES ---
app.get('/notificaciones', async (req, res) => {
  try {
    const [rows] = await pool.query('SELECT * FROM notificaciones');
    res.json({ ok: true, servicio: 'notificaciones', vm: process.env.APP_NAME, data: rows });
  } catch(err) { res.status(500).json({ ok: false, error: err.message }); }
});

// --- HEALTH Y ROOT ---
app.get('/', (req, res) => {
  res.json({ servicio: 'microservicios', vm: process.env.APP_NAME, estado: 'operativo' });
});

app.get('/health', (req, res) => {
  res.json({ status: 'ok', vm: process.env.APP_NAME, puerto: PORT });
});

app.listen(PORT, '0.0.0.0', () => {
  console.log('App corriendo en puerto ' + PORT + ' — ' + process.env.APP_NAME);
});
```

### Lanzar con PM2
```bash
cd ~/app

# Iniciar la app con PM2 y asignarle un nombre identificable
pm2 start index.js --name app2-server158

# Verificar que está corriendo
pm2 list
```

Resultado esperado:
```
┌────┬──────────────────┬────────┬───────────┐
│ id │ name             │ mode   │ status    │
├────┼──────────────────┼────────┼───────────┤
│  0 │ app2-server158   │ fork   │ online    │
└────┴──────────────────┴────────┴───────────┘
```

```bash
# Configurar PM2 para que inicie automáticamente al reiniciar el servidor
pm2 startup
# Copiar y ejecutar el comando que PM2 muestra en pantalla

# Guardar la lista de procesos actual
pm2 save
```

### Verificar endpoints
```bash
# Verificar que la app responde correctamente en cada microservicio
curl http://localhost:3001/
curl http://localhost:3001/health
curl http://localhost:3001/usuarios
curl http://localhost:3001/pagos
curl http://localhost:3001/notificaciones
```

Resultado esperado en `/usuarios`:
```json
{"ok":true,"servicio":"usuarios","vm":"app2-server158","data":[...]}
```

### Firewall UFW
```bash
# Denegar todo el tráfico entrante por defecto (principio de mínimo privilegio)
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Permitir solo los puertos necesarios
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 3001/tcp  # App Node.js
sudo ufw allow 9100/tcp  # Node Exporter (métricas para Prometheus)

# Activar firewall
sudo ufw --force enable
sudo ufw status verbose
```

### Hardening SSH
```bash
# Deshabilitar login como root — elimina el usuario más atacado
sudo sed -i 's/^#*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config

# Limitar intentos de autenticación fallidos a 3
sudo sed -i 's/^#*MaxAuthTries.*/MaxAuthTries 3/' /etc/ssh/sshd_config

sudo systemctl restart ssh

# Verificar que los cambios fueron aplicados
grep -E "^(PermitRootLogin|MaxAuthTries)" /etc/ssh/sshd_config
```

Resultado esperado:
```
PermitRootLogin no
MaxAuthTries 3
```

### Node Exporter (métricas para Prometheus)
```bash
# Instalar agente que expone métricas del SO en el puerto 9100
sudo apt install prometheus-node-exporter -y
sudo systemctl enable prometheus-node-exporter
sudo systemctl start prometheus-node-exporter

# Verificar que las métricas están disponibles
curl http://localhost:9100/metrics | head -5
```

---

## Server-159 — MariaDB Master

### Instalación
```bash
sudo apt update
sudo apt install mariadb-server -y
sudo systemctl enable mariadb
sudo systemctl start mariadb
sudo systemctl status mariadb
```

### Hardening inicial
```bash
# Script interactivo que asegura la instalación de MariaDB:
# - Establece contraseña root
# - Elimina usuarios anónimos
# - Deshabilita acceso root remoto
# - Elimina base de datos de prueba
sudo mysql_secure_installation
```

### Configuración de acceso remoto y replicación
```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

```ini
# Escuchar en todas las interfaces para permitir acceso remoto desde las apps
bind-address   = 0.0.0.0

# ID único del servidor Master (debe ser diferente al Slave)
server-id      = 1

# Habilitar binary log — registra todos los cambios para replicación
log_bin        = /var/log/mysql/mysql-bin.log

# Solo replicar esta base de datos
binlog_do_db   = proyecto_db
```

```bash
# Crear directorio para los archivos de binlog (no existe por defecto)
sudo mkdir -p /var/log/mysql
sudo chown mysql:mysql /var/log/mysql

sudo systemctl restart mariadb
```

### Crear BD, tablas, usuarios y datos
```bash
sudo mysql -u root -p
```

```sql
-- Base de datos del proyecto
CREATE DATABASE proyecto_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Usuario para las apps (server-157 y server-158)
-- El '%' permite conexión desde cualquier IP — las apps se conectan desde 157 y 158
CREATE USER 'appuser'@'%' IDENTIFIED BY 'App1234!';
GRANT ALL PRIVILEGES ON proyecto_db.* TO 'appuser'@'%';

-- Usuario exclusivo para replicación — solo puede conectarse desde server-160
CREATE USER 'replicador'@'192.168.100.160' IDENTIFIED BY 'Repl1234!';
GRANT REPLICATION SLAVE ON *.* TO 'replicador'@'192.168.100.160';

FLUSH PRIVILEGES;

USE proyecto_db;

-- Tabla de usuarios del sistema
CREATE TABLE usuarios (
  id INT AUTO_INCREMENT PRIMARY KEY,
  nombre VARCHAR(100) NOT NULL,
  email VARCHAR(150) UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Tabla de pagos — referencia a usuarios con FOREIGN KEY
CREATE TABLE pagos (
  id INT AUTO_INCREMENT PRIMARY KEY,
  usuario_id INT NOT NULL,
  monto DECIMAL(10,2) NOT NULL,
  concepto VARCHAR(200),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (usuario_id) REFERENCES usuarios(id)
);

-- Tabla de notificaciones del sistema
CREATE TABLE notificaciones (
  id INT AUTO_INCREMENT PRIMARY KEY,
  usuario_id INT NOT NULL,
  mensaje VARCHAR(300) NOT NULL,
  leida BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (usuario_id) REFERENCES usuarios(id)
);

-- Datos de prueba
INSERT INTO usuarios (nombre, email) VALUES
  ('Marco Sanchez', 'marco@proyecto.com'),
  ('Jose Miranda', 'jose@proyecto.com'),
  ('Ana Lopez', 'ana@proyecto.com');

INSERT INTO pagos (usuario_id, monto, concepto) VALUES
  (1, 150.00, 'Mensualidad enero'),
  (2, 200.50, 'Mensualidad febrero'),
  (3, 175.00, 'Mensualidad marzo');

INSERT INTO notificaciones (usuario_id, mensaje) VALUES
  (1, 'Tu pago fue registrado exitosamente'),
  (2, 'Nuevo usuario registrado'),
  (3, 'Bienvenido al sistema');

-- Verificar datos
SELECT * FROM usuarios;
SELECT * FROM pagos;
SELECT * FROM notificaciones;
SHOW DATABASES;
EXIT;
```

### Obtener posición del binlog para el Slave
```bash
sudo mysql -u root -p
```

```sql
-- Muestra el archivo y posición actual del binlog
-- ANOTAR estos valores — son necesarios para configurar el Slave
SHOW MASTER STATUS;
EXIT;
```

Resultado esperado:
```
+------------------+----------+--------------+
| File             | Position | Binlog_Do_DB |
+------------------+----------+--------------+
| mysql-bin.000001 |     2952 | proyecto_db  |
+------------------+----------+--------------+
```

### Firewall y hardening SSH
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 3306/tcp  # MariaDB — accesible desde las apps
sudo ufw allow 9100/tcp  # Node Exporter
sudo ufw --force enable
sudo ufw status verbose

# SSH hardening
sudo sed -i 's/^#*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo sed -i 's/^#*MaxAuthTries.*/MaxAuthTries 3/' /etc/ssh/sshd_config
sudo systemctl restart ssh
grep -E "^(PermitRootLogin|MaxAuthTries)" /etc/ssh/sshd_config
```

### Node Exporter
```bash
sudo apt install prometheus-node-exporter -y
sudo systemctl enable prometheus-node-exporter
sudo systemctl start prometheus-node-exporter
curl http://localhost:9100/metrics | head -5
```

---

## Server-160 — MariaDB Slave

### Instalación
```bash
sudo apt update
sudo apt install mariadb-server -y
sudo systemctl enable mariadb
sudo systemctl start mariadb
```

### Configuración del Slave
```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

```ini
# Escuchar en todas las interfaces
bind-address   = 0.0.0.0

# ID único del Slave — debe ser diferente al Master (que tiene server-id=1)
server-id      = 2

# Archivo donde el Slave almacena los eventos recibidos del Master
relay-log      = /var/log/mysql/mysql-relay-bin.log
```

```bash
# Crear directorio para relay-log
sudo mkdir -p /var/log/mysql
sudo chown mysql:mysql /var/log/mysql

sudo systemctl restart mariadb
```

### Enlazar con el Master e iniciar replicación
```bash
sudo mysql -u root
```

```sql
-- Configurar la conexión al Master
-- Reemplazar MASTER_LOG_FILE y MASTER_LOG_POS con los valores obtenidos en SHOW MASTER STATUS
CHANGE MASTER TO
  MASTER_HOST='192.168.100.159',
  MASTER_USER='replicador',
  MASTER_PASSWORD='Repl1234!',
  MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS=2952;

-- Iniciar el proceso de replicación
START SLAVE;

-- Verificar estado de la replicación
SHOW SLAVE STATUS\G
EXIT;
```

### Verificar replicación activa
```bash
sudo mysql -u root -e "SHOW SLAVE STATUS\G" | grep -E "Slave_IO_Running|Slave_SQL_Running|Seconds_Behind|Last_Error"
```

Resultado esperado:
```
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
Seconds_Behind_Master: 0
Last_Error:
```

> Si `Slave_IO_Running` o `Slave_SQL_Running` muestran `No`, revisar `Last_Error` para diagnosticar el problema.

### Firewall y Node Exporter
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 3306/tcp
sudo ufw allow 9100/tcp
sudo ufw --force enable

sudo apt install prometheus-node-exporter -y
sudo systemctl enable prometheus-node-exporter
sudo systemctl start prometheus-node-exporter
```

---

## Server-161 — Prometheus + Grafana

### Instalación de Prometheus y Node Exporter
```bash
sudo apt update

# Prometheus — servidor de monitoreo que recolecta métricas por scraping
# Node Exporter — agente que expone métricas del SO local
sudo apt install prometheus prometheus-node-exporter -y

sudo systemctl enable prometheus
sudo systemctl enable prometheus-node-exporter
sudo systemctl start prometheus
sudo systemctl start prometheus-node-exporter
sudo systemctl status prometheus
```

### Configurar targets en Prometheus
```bash
sudo nano /etc/prometheus/prometheus.yml
```

```yaml
global:
  scrape_interval: 15s       # Recolectar métricas cada 15 segundos
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'servidores-grupo6'
    static_configs:
      - targets:
        - '192.168.100.157:9100'  # server-157 app1
        - '192.168.100.158:9100'  # server-158 app2
        - '192.168.100.159:9100'  # server-159 bdmaster
        - '192.168.100.160:9100'  # server-160 bdslave
        - '192.168.100.161:9100'  # server-161 grafana
```

```bash
sudo systemctl restart prometheus
sudo systemctl status prometheus
```

### Instalación de Grafana
```bash
# Instalar dependencias previas
sudo apt install -y apt-transport-https software-properties-common wget

# Crear directorio para la clave GPG del repositorio oficial de Grafana
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

# Agregar repositorio estable de Grafana
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

sudo apt update
sudo apt install grafana -y
sudo systemctl daemon-reload
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server
```

### Firewall
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 3000/tcp  # Grafana
sudo ufw allow 9090/tcp  # Prometheus
sudo ufw allow 9100/tcp  # Node Exporter
sudo ufw --force enable
sudo ufw status verbose
```

### Acceso a Grafana
Grafana corre en el puerto 3000 de server-161 (IP interna). Para acceder desde tu PC usar tunnel SSH:

```bash
# En una terminal local — redirige el puerto 3000 de server-161 a tu localhost
ssh -L 3000:192.168.100.161:3000 usrproxy@201.131.45.42
```

Luego abrir en el navegador: `http://localhost:3000`

- Usuario: `admin`
- Contraseña inicial: `admin` (cambiar al primer ingreso)

### Configurar Prometheus como Data Source
1. Ir a **Connections → Data Sources → Add data source**
2. Seleccionar **Prometheus**
3. URL: `http://localhost:9090`
4. Clic en **Save & Test**

Resultado esperado: `Successfully queried the Prometheus API`

### Importar Dashboard
1. Ir a **Dashboards → New → Import**
2. En "Import via grafana.com" escribir: `1860`
3. Clic en **Load → Import**

> El dashboard **Node Exporter Full** muestra CPU, RAM, disco y red de todos los servidores monitoreados.

---

## Scripts de Automatización (server-158)

### Script 1 — Health Check
Verifica que todos los servicios del grupo estén respondiendo correctamente.

```bash
mkdir -p ~/scripts
nano ~/scripts/health_check.sh
```

```bash
#!/bin/bash
echo '=== HEALTH CHECK GRUPO 6 ==='
echo "Fecha: $(date)"
echo ''

check_endpoint() {
  local name=$1 url=$2
  if curl -s --max-time 3 "$url" | grep -q 'ok'; then
    echo "[OK]  $name -> $url"
  else
    echo "[FAIL] $name -> $url"
  fi
}

echo '--- Apps ---'
check_endpoint 'App1 health' 'http://192.168.100.157:3001/health'
check_endpoint 'App2 health' 'http://192.168.100.158:3001/health'

echo ''
echo '--- PM2 local ---'
pm2 list

echo ''
echo '--- MariaDB Master ---'
if mysql -h 192.168.100.159 -u appuser -pApp1234! proyecto_db -e 'SELECT 1;' &>/dev/null; then
  echo '[OK]  MariaDB Master (192.168.100.159) accesible'
else
  echo '[FAIL] MariaDB Master NO responde'
fi
```

```bash
chmod +x ~/scripts/health_check.sh
~/scripts/health_check.sh
```

### Script 2 — Backup de Base de Datos
Genera un volcado de la BD, verifica su integridad y rota los backups antiguos.

```bash
nano ~/scripts/backup_db.sh
```

```bash
#!/bin/bash
BACKUP_DIR=~/backups
DATE=$(date +%Y%m%d_%H%M%S)
FILE="$BACKUP_DIR/proyecto_db_$DATE.sql"

mkdir -p $BACKUP_DIR

echo "Iniciando backup: $FILE"
mysqldump -h 192.168.100.159 -u appuser -pApp1234! proyecto_db > $FILE

# Verificar que el archivo contiene el marcador de fin de dump
if grep -q 'Dump completed' $FILE; then
  echo '[OK]  Backup completado y verificado'
  gzip $FILE
  echo "Archivo: ${FILE}.gz"
else
  echo '[ERROR] Backup incompleto — eliminando archivo corrupto'
  rm -f $FILE
  exit 1
fi

# Conservar solo los últimos 7 backups (rotación automática)
ls -t $BACKUP_DIR/*.sql.gz 2>/dev/null | tail -n +8 | xargs rm -f
echo "Backups actuales: $(ls $BACKUP_DIR/*.sql.gz 2>/dev/null | wc -l)"
```

```bash
chmod +x ~/scripts/backup_db.sh

# Programar ejecución automática todos los días a las 02:00
crontab -e
# Agregar esta línea:
# 0 2 * * * /home/adming6/scripts/backup_db.sh >> /home/adming6/backups/backup.log 2>&1
```

### Script 3 — Deploy
Actualiza la aplicación desde Git y reinicia PM2.

```bash
nano ~/scripts/deploy.sh
```

```bash
#!/bin/bash
APP_DIR=~/app

echo '=== DEPLOY ==='
echo "Servidor: $(hostname) — $(date)"

cd $APP_DIR

echo 'Actualizando desde Git...'
git pull origin main

echo 'Instalando dependencias...'
npm install --production

echo 'Reiniciando PM2...'
pm2 restart all

sleep 2
pm2 list
echo '=== Deploy completado ==='
```

```bash
chmod +x ~/scripts/deploy.sh
```

### Script 4 — Menú Interactivo
Interfaz de administración accesible por SSH para gestionar todos los servicios.

```bash
nano ~/menu.sh
```

```bash
#!/bin/bash
while true; do
  clear
  echo '======================================='
  echo '   ADMINISTRACIÓN — GRUPO 6 SIS313'
  echo "   $(hostname) | $(date '+%Y-%m-%d %H:%M')"
  echo '======================================='
  echo '  1) Health Check de todos los servicios'
  echo '  2) Estado de PM2'
  echo '  3) Reiniciar apps con PM2'
  echo '  4) Backup de Base de Datos'
  echo '  5) Deploy (actualizar desde Git)'
  echo '  6) Ver logs de PM2'
  echo '  7) Ver backups disponibles'
  echo '  0) Salir'
  echo '======================================='
  read -p '  Opcion: ' opt
  case $opt in
    1) ~/scripts/health_check.sh ; read -p 'Presiona ENTER...' ;;
    2) pm2 list ; read -p 'Presiona ENTER...' ;;
    3) pm2 restart all ; read -p 'Presiona ENTER...' ;;
    4) ~/scripts/backup_db.sh ; read -p 'Presiona ENTER...' ;;
    5) ~/scripts/deploy.sh ; read -p 'Presiona ENTER...' ;;
    6) pm2 logs --lines 30 ; read -p 'Presiona ENTER...' ;;
    7) ls -lh ~/backups/*.sql.gz 2>/dev/null || echo 'Sin backups' ; read -p 'Presiona ENTER...' ;;
    0) echo 'Saliendo...' ; exit 0 ;;
    *) echo 'Opcion invalida' ;;
  esac
done
```

```bash
chmod +x ~/menu.sh
~/menu.sh
```

---

## Verificación Final del Sistema

### Probar balanceo de carga
```bash
# Hacer varias peticiones y verificar que alterna entre app1 y app2
# El campo "vm" en la respuesta indica qué servidor respondió
for i in {1..6}; do
  curl -s http://201.131.45.42/health | grep vm
done
```

Resultado esperado (alternando):
```
"vm":"app1-server157"
"vm":"app2-server158"
"vm":"app1-server157"
"vm":"app2-server158"
```

### Probar failover
```bash
# En server-157 — detener la app
pm2 stop all

# Desde cualquier cliente — todas las respuestas deben venir de app2
curl http://201.131.45.42/usuarios

# En server-157 — restaurar
pm2 start all
```

### Verificar replicación
```bash
# En server-159 (Master) — insertar dato de prueba
sudo mysql -u root -p -e "INSERT INTO proyecto_db.usuarios (nombre,email) VALUES ('Test','test@rep.com');"

# En server-160 (Slave) — verificar que se replicó automáticamente
sudo mysql -u root -e "SELECT * FROM proyecto_db.usuarios WHERE email='test@rep.com';"
```

---

*Grupo 6 — SIS313 — USFX — Semestre 1/2026*
