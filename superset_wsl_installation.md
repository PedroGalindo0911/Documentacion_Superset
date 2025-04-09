
# 📊 Instalación de Apache Superset en Windows usando WSL (Ubuntu)

## 🐧 Paso 1: Instalación de WSL y Ubuntu

1. Abre la **Microsoft Store** e instala la aplicación **Ubuntu**.
2. Una vez instalada, ábrela. Se mostrará una ventana de consola indicando que WSL no ha sido habilitado.
3. Cierra la ventana, abre **PowerShell o CMD como administrador** y ejecuta:

   ```bash
   wsl --install
   ```

4. Al finalizar la instalación, **reinicia tu dispositivo**.
5. Abre la aplicación de **Ubuntu** nuevamente. Se te pedirá crear un **usuario y contraseña** (elige cualquier nombre y clave).
6. Ejecuta las siguientes actualizaciones básicas:

   ```bash
   sudo apt update -y && sudo apt upgrade -y
   ```

## 📦 Paso 2: Instalación de dependencias

```bash
sudo apt-get install build-essential libssl-dev libffi-dev python3-dev python3-pip libsasl2-dev libldap2-dev default-libmysqlclient-dev python3.10-venv unixodbc unixodbc-dev msodbcsql18
```

## 🗂️ Paso 3: Configuración del entorno de Superset

```bash
sudo mkdir /app
sudo chown $USER /app
cd /app
mkdir superset
cd superset
python3 -m venv superset_env
. superset_env/bin/activate
pip install --upgrade setuptools pip
```

## 🧩 Paso 4: Instalación de Superset y dependencias

```bash
pip install pillow
pip install apache-superset
```

Genera tu **secret key**:

```bash
openssl rand -base64 42
```

Guarda esa llave para el siguiente paso.

## ⚙️ Paso 5: Configuración de Superset

```bash
touch superset_config.py
export SUPERSET_CONFIG_PATH=/app/superset/superset_config.py
```

Edita el archivo con:

```bash
nano superset_config.py
```

Pega lo siguiente (reemplaza `'YOUR_OWN_RANDOM_GENERATED_SECRET_KEY'` por tu llave secreta):

```python
# Superset specific config
ROW_LIMIT = 5000

# Flask App Builder configuration
SECRET_KEY = 'YOUR_OWN_RANDOM_GENERATED_SECRET_KEY'

SQLALCHEMY_DATABASE_URI = 'sqlite:////app/superset/superset.db?check_same_thread=false'

TALISMAN_ENABLED = False
WTF_CSRF_ENABLED = False

MAPBOX_API_KEY = ' '
```

## 🛠️ Paso 6: Inicializar Superset

```bash
export FLASK_APP=superset
superset db upgrade
superset fab create-admin
superset init
```

## 🖥️ Paso 7: Crear script de ejecución

```bash
nano run_superset.sh
```

Pega esto:

```bash
#!/bin/bash
export SUPERSET_CONFIG_PATH=/app/superset/superset_config.py
. /app/superset/superset_env/bin/activate
gunicorn \
  -w 10 \
  -k gevent \
  --timeout 120 \
  -b 0.0.0.0:8088 \
  --limit-request-line 0 \
  --limit-request-field_size 0 \
  --statsd-host localhost:8125 \
  "superset.app:create_app()"
```

Guarda y dale permisos:

```bash
chmod +x run_superset.sh
sh run_superset.sh
```

## 🔄 Paso 8: Crear servicio para iniciar Superset automáticamente

```bash
sudo nano /etc/systemd/system/superset.service
```

Pega:

```ini
[Unit]
Description = Apache Superset Webserver Daemon
After = network.target

[Service]
PIDFile = /app/superset/superset-webserver.PIDFile
Environment=SUPERSET_HOME=/app/superset
Environment=PYTHONPATH=/app/superset
WorkingDirectory = /app/superset
ExecStart = /app/superset/run_superset.sh
ExecStop = /bin/kill -s TERM $MAINPID

[Install]
WantedBy=multi-user.target
```

Activa el servicio:

```bash
sudo systemctl daemon-reload
sudo systemctl enable superset.service
sudo systemctl start superset.service
```

## 🔌 Paso 9: Conectar a una base de datos SQL Server

1. Verifica si tienes el driver ODBC:

   ```bash
   odbcinst -q -d | grep "ODBC Driver 18 for SQL Server"
   ```

2. Obtén la IP del host de Windows:

   ```bash
   cat /etc/resolv.conf | grep nameserver | awk '{print $2}'
   ```

3. En Superset, ve a **Bases de datos > Agregar**. Si no aparece SQL Server, elige "Otros".
4. Usa una URL similar a esta (reemplaza los valores correspondientes):

   ```
   mssql+pyodbc://user:password@ipwsl:puerto/S7BDG_Demo?driver=ODBC+Driver+18+for+SQL+Server&TrustServerCertificate=yes
   ```

5. En **Opciones Avanzadas > SQL Lab**, activa la opción *Allow DDL and DML*.

## 🌐 Paso 10: Habilitar Embedding de Dashboards

1. En Ubuntu:

   ```bash
   cd /app/superset
   nano superset_config.py
   ```

2. Agrega al final del archivo:

   ```python
   PUBLIC_ROLE_LIKE = "Gamma"
   ```

3. Reinicia Superset:

   ```bash
   sudo systemctl restart superset
   ```

4. En Superset:
   - Ve a **Seguridad > Roles**.
   - Elimina el rol `Public`.
   - Clona el rol `Gamma`, renómbralo a `Public`.
   - Agrega permisos de acceso a los gráficos que deseas embeder.
5. Copia el enlace del dashboard (modo pantalla completa) y pégalo en tu HTML:

   ```html
   <iframe src="http://localhost:8088/superset/dashboard/p/9X10dj58W7v/" allowfullscreen=""></iframe>
   ```
