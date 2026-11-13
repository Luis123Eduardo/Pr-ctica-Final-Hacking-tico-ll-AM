# 🎯 PRÁCTICA 1: INGENIERÍA SOCIAL (Valor 5 puntos)

**Objetivo:** Engañar a una persona de confianza para obtener acceso a su red social.

---

### 📌 Antes de grabar (Preparación)

**1. Elegir la víctima:** Un amigo o familiar (no pareja) que acepte participar.

**2. Tener listo el entorno:**

- Kali Linux encendido y con Internet.
- Tu máquina víctima (el celular o PC del amigo) lista para recibir el enlace.
- El guion de la charla de concientización a la mano.

---

### 🎬 EMPIEZA EL VIDEO

*(En cámara: "Hola, soy Luis Eduardo Méndez, matrícula 2025-0826. Hoy voy a hacer la práctica de Ingeniería Social del ITLA. El objetivo es demostrar lo fácil que puede ser engañar a un usuario para robarle sus credenciales de una red social, usando una herramienta de phishing ético.")*

---

### 🚀 PASO 1: Instalar Zphisher en Kali

**En la terminal de Kali, escribe:**

```bash
git clone --depth=1 https://github.com/htr-tech/zphisher.git
```

*Explicación: "Este comando descarga la herramienta Zphisher desde su repositorio oficial de GitHub. Es una herramienta de código abierto para crear páginas de phishing educativas."*

```bash
cd zphisher
```

*Explicación: "Entro a la carpeta de Zphisher."*

```bash
bash zphisher.sh
```

*Explicación: "Ejecuto el script de Zphisher. La primera vez descargará las dependencias necesarias como PHP y curl."*

---

### 🎯 PASO 2: Elegir la plantilla de la red social

**Zphisher mostrará el menú "Select An Attack For Your Victim" con la lista de plantillas:**

```
[01] Facebook
[02] Instagram
[03] Google
[04] Microsoft
[05] Netflix
[06] PayPal
[07] Steam
[08] Twitter
[09] PlayStation
[10] Tiktok
...
```

**Escribe `2` (Instagram) y presiona Enter.**

*Explicación: "Elijo la plantilla de Instagram porque es la red social que mi víctima usa."*

**Luego te mostrará las variantes de Instagram:**

```
[01] Traditional Login Page (Página de inicio de sesión tradicional)
[02] Auto Followers Login Page (Página de inicio de sesión con seguidores automáticos)
[03] 1000 Followers Login Page (Página de inicio de sesión con 1000 seguidores)
[04] Blue Badge Verify Login Page (Página de inicio de sesión con verificación de insignia azul)
```

**Escribe `1` (Traditional Login Page) y presiona Enter.**

*Explicación: "Elijo la página de inicio de sesión tradicional porque es la que se ve más realista."*

---

### 🌐 PASO 3: Elegir el método de exposición

**Después de la plantilla, Zphisher mostrará el menú de túneles:**

```
[01] Localhost (Local)
[02] Cloudflared [Auto Detects] (Cloudflared [Detección automática])
[03] LocalXpose [NEW! Max 15Min] (LocalXpose [¡NUEVO! Máx. 15 min])
```

**Escribe `2` (Cloudflared) y presiona Enter.**

*Explicación: "Elijo Cloudflared porque genera un enlace público que mi víctima puede abrir desde su celular, sin importar la red en la que esté."*

**Zphisher preguntará:**

```
[?] Do You Want A Custom Port [y/N]: (¿Quieres un puerto personalizado?)
```

**Escribe `n` y presiona Enter.**

*Explicación: "Le digo que no quiero un puerto personalizado, para que use el puerto por defecto (8080)."*

**Y por último preguntará:**

```
[?] Do you want to change Mask URL? [y/N]: (¿Quieres cambiar la URL enmascarada?)
```

**Escribe `n` y presiona Enter.**

*Explicación: "Le digo que no, porque no necesito enmascarar la URL."*

---

### 🔗 PASO 4: Obtener el enlace y enviarlo a la víctima

**Zphisher generará el enlace público. Verás algo como:**

```
[-] URL 1 : https://discussed-gratuit-altered-prizes.trycloudflare.com
[-] URL 2 : https://
[-] URL 3 : https://get-unlimited-followers-for-instagram@
[-] Waiting for Login Info, Ctrl + C to exit ... (Esperando información de inicio de sesión, Ctrl + C para salir)
```

**Copia la URL 1 (el enlace de Cloudflared).**

*Explicación: "Copio el enlace público que generó Cloudflared. Este es el enlace que le voy a enviar a mi víctima."*

⚠️ **Reglas del túnel (para que el enlace no se muera):**

- El terminal de Kali con "Waiting for Login Info" debe quedarse ABIERTO todo el tiempo — ese terminal es el túnel. Cerrarlo o presionar Ctrl+C = el enlace muere.
- Kali necesita Internet mientras la víctima usa la página.
- Cada vez que reinicias Zphisher se genera una URL NUEVA; la vieja nunca revive.
- Prueba el enlace TÚ primero (en el navegador de Kali o en tu propio celular) antes de mandárselo a la víctima.
- Envíalo por WhatsApp como mensaje de texto normal (que quede clicable) y verifica que el mensaje tenga TODAS las palabras de la URL.

**Envíale un mensaje convincente por WhatsApp:**

> *"Mira, necesito que me ayudes con esta encuesta rápida de Instagram, es para un trabajo del ITLA. Entra aquí: \[pegar el enlace\]"*

---

### 🎣 PASO 5: Capturar las credenciales

**Cuando tu víctima abra el enlace y escriba su usuario y contraseña, Zphisher capturará los datos en tu terminal de Kali.**

**Verás algo como:**

```
[-] Victim IP Found ! (¡IP de la víctima encontrada!)
[-] Victim's IP : 186.149.68.253 (IP de la víctima)
[-] Saved in : auth/ip.txt (Guardado en)

[-] Login info Found !! (¡Información de inicio de sesión encontrada!)
[-] Account : hola (Cuenta)
[-] Password : 902384 (Contraseña)
[-] Saved in : auth/usernames.dat (Guardado en)
```

*Explicación: "En mi terminal puedo ver las credenciales que mi víctima ingresó: su cuenta y su contraseña. Esto demuestra lo fácil que es caer en un phishing."*

---

### 🎬 PASO 6: Grabar el resultado y dar la charla

**1. Muestra en pantalla las credenciales capturadas.**

**2. Entra a la cuenta de la víctima (con su permiso) para demostrar el hackeo.**

**3. Graba la charla de concientización (obligatoria):**

> *"Soy estudiante del tecnólogo en seguridad del ITLA, estoy cursando la materia de Hacker Ético II con el Profesor Diógenes Jiménez. Estoy realizando una práctica del maestro donde debemos demostrar la facilidad con la que se puede hackear a un usuario, y lo vulnerable que somos todos ante este tipo de ataques.*
>
> *Yo realicé mi práctica contigo. Lo que hice fue lo siguiente: creé una página web falsa que era idéntica a la pantalla de inicio de sesión de Instagram. Esa página no era de Instagram, era mía, pero se veía exactamente igual. Luego te envié un enlace por WhatsApp diciéndote que me ayudaras con una encuesta, y cuando entraste y escribiste tu usuario y contraseña, esos datos me llegaron directamente a mí.*
>
> *Con esa información pude entrar a tu cuenta de Instagram, y aquí está la prueba. Quiero que veas que no fue nada complicado, no usé programas avanzados ni nada raro: solo un enlace y un poco de psicología.*
>
> *Para evitarlo, quiero decirte que:*
>
> 1. *Nunca ingreses tu contraseña desde enlaces que te lleguen por WhatsApp, correo o redes sociales, aunque parezcan de alguien de confianza. Siempre verifica la URL real.*
> 2. *Activa la verificación en dos pasos (2FA) en Instagram. Así, aunque alguien tenga tu contraseña, no podrá entrar sin el código de tu teléfono.*
> 3. *Desconfía de mensajes que te piden ayuda urgente o que te ofrecen algo a cambio de un clic.*
>
> *Esta práctica es para concientizar sobre lo fácil que es caer en un phishing. No se trata de asustarte, sino de que estés atento la próxima vez que te llegue un enlace sospechoso."*

---

### 🎓 CIERRE

*(En cámara: "Con esto concluyo la práctica de Ingeniería Social. Aprendí a usar Zphisher para crear una página de phishing, enviar el enlace a una víctima y capturar sus credenciales. La práctica se completó exitosamente y sirvió para concientizar a la persona sobre los riesgos del phishing.")*
