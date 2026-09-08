#Criptografía aplicada e identidad digital

**Módulo:** Desarrollo seguro, criptografía e IAM
**Semana:** 3 — Criptografía e identidad
**Modalidad:** laboratorio guiado individual
**Entorno:** Kali Linux
**Duración estimada:** 6 horas (2 de sesión guiada + 4 de trabajo autónomo)

---

## 1. Propósito de la actividad

En esta actividad el estudiante aplica, de forma práctica y con la línea de comandos de Kali Linux, los conceptos de criptografía moderna: cifrado simétrico y asimétrico, funciones hash, firma digital y certificados digitales. Todas las operaciones se realizan con **OpenSSL**, la biblioteca criptográfica que viene preinstalada en Kali. En la segunda parte, el estudiante analiza cómo la aplicación vulnerable de las semanas anteriores (OWASP Juice Shop) almacena las contraseñas y propone la corrección adecuada, conectando así la criptografía con el control de acceso.

Al terminar, el estudiante estará en capacidad de:

- Explicar la diferencia entre cifrado simétrico, cifrado asimétrico y funciones hash, y cuándo se usa cada uno.
- Cifrar y descifrar archivos con AES-256 usando OpenSSL.
- Generar un par de claves RSA, calcular y verificar hashes, y firmar y verificar un archivo.
- Crear una Autoridad Certificadora (CA) propia y emitir un certificado digital para servir contenido por HTTPS.
- Evaluar el almacenamiento de contraseñas de una aplicación y proponer un esquema seguro (Argon2/bcrypt) según la guía de OWASP.

---

## 2. Requisitos previos

| Requisito | Detalle |
|---|---|
| Sistema operativo | **Kali Linux** (instalación nativa o máquina virtual) |
| OpenSSL | Preinstalado en Kali. Se verifica en el Paso 1 |
| Navegador | Firefox ESR (preinstalado) |
| Editor de texto | `mousepad` o `nano` (preinstalados) |
| Docker | Solo para la Parte C opcional (ya instalado en la semana 2) |
| Lecturas previas | [OWASP Top 10:2021 – A02 Fallas Criptográficas (español)](https://owasp.org/Top10/2021/es/A02_2021-Cryptographic_Failures/) y capítulos de hash y cifrado de [Criptografía para Ingenier@s (Jorge Ramió)](https://www.criptored.es/paginas/Criptograf%C3%ADa%20para%20Ingenier@s.html) |

> **Nota:** todos los comandos se ejecutan en la **terminal de Kali** (`Ctrl+Alt+T`). En esta actividad casi ningún comando requiere `sudo`, salvo los que se indican explícitamente. Los archivos que empiezan por `clave` o terminan en `.key` son claves privadas: **no deben compartirse ni subirse a GitHub**.

---

## 3. Conceptos mínimos antes de empezar

- **Cifrado simétrico:** usa **la misma clave** para cifrar y descifrar (ejemplo: AES). Es rápido; el reto es compartir la clave de forma segura.
- **Cifrado asimétrico:** usa un **par de claves**, una pública y una privada (ejemplo: RSA). Lo que cifra la pública solo lo descifra la privada, y viceversa.
- **Función hash:** produce una huella de tamaño fijo (ejemplo: SHA-256) a partir de cualquier dato. No es reversible y sirve para verificar integridad. **MD5 y SHA-1 se consideran inseguros** y no deben usarse.
- **Firma digital:** hash del documento cifrado con la clave privada del autor; cualquiera puede verificarla con la clave pública. Aporta integridad, autenticidad y no repudio.
- **Certificado digital:** documento que asocia una clave pública con una identidad (por ejemplo, un dominio) y va firmado por una **Autoridad Certificadora (CA)**. Es la base de HTTPS (estándar X.509).

---

## 4. Paso a paso

### Paso 1. Preparar el entorno

1. El estudiante verifica que OpenSSL está disponible y anota la versión:

   ```bash
   openssl version
   ```

   Debe responder algo como `OpenSSL 3.x`. Si no estuviera instalado: `sudo apt update && sudo apt install -y openssl`.

2. Crea y entra en la carpeta de trabajo:

   ```bash
   mkdir -p ~/semana3-cripto
   cd ~/semana3-cripto
   ```

3. Crea un archivo de prueba con un mensaje propio (debe incluir el nombre del estudiante para que las evidencias sean personales):

   ```bash
   echo "Mensaje confidencial de <NOMBRE DEL ESTUDIANTE> - Semana 3" > mensaje.txt
   cat mensaje.txt
   ```

---

### Parte A · Cifrado simétrico, hash y firma

### Paso 2. Cifrado simétrico con AES-256

1. Cifra el archivo `mensaje.txt` con AES-256. OpenSSL pedirá una contraseña de la que derivará la clave:

   ```bash
   openssl enc -aes-256-cbc -pbkdf2 -salt -in mensaje.txt -out mensaje.enc
   ```

   > La opción `-pbkdf2` deriva la clave de forma robusta a partir de la contraseña y `-salt` añade aleatoriedad. Sin ellas el cifrado sería débil; forman parte de las buenas prácticas.

2. Observa que el archivo cifrado no es legible:

   ```bash
   cat mensaje.enc
   ```

3. Descifra el archivo (pedirá la misma contraseña) y comprueba que el contenido coincide con el original:

   ```bash
   openssl enc -d -aes-256-cbc -pbkdf2 -in mensaje.enc -out mensaje_descifrado.txt
   diff mensaje.txt mensaje_descifrado.txt && echo "IGUALES: descifrado correcto"
   ```

> **Evidencia 1:** captura de la terminal donde se vea el cifrado, el contenido ilegible de `mensaje.enc` y el mensaje `IGUALES: descifrado correcto`.

### Paso 3. Funciones hash e integridad

1. Calcula el hash SHA-256 del archivo original:

   ```bash
   sha256sum mensaje.txt
   ```

2. Modifica una sola letra del archivo y vuelve a calcular el hash para comprobar que cambia por completo (efecto avalancha):

   ```bash
   echo "Mensaje MODIFICADO de <NOMBRE> - Semana 3" > mensaje_v2.txt
   sha256sum mensaje.txt mensaje_v2.txt
   ```

3. Compara con un algoritmo obsoleto y anota la diferencia de longitud (MD5 produce 128 bits; SHA-256, 256 bits):

   ```bash
   md5sum mensaje.txt
   ```

   En el informe el estudiante explica **por qué MD5 no debe usarse** hoy (colisiones demostradas).

> **Evidencia 2:** captura donde se vean los dos hashes SHA-256 (original y modificado) y se aprecie que son totalmente distintos.

### Paso 4. Par de claves RSA y firma digital

1. Genera una clave privada RSA de 3072 bits:

   ```bash
   openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:3072 -out clave_privada.key
   ```

2. Extrae la clave pública correspondiente:

   ```bash
   openssl pkey -in clave_privada.key -pubout -out clave_publica.pem
   ```

3. Firma el archivo `mensaje.txt` con la clave privada:

   ```bash
   openssl dgst -sha256 -sign clave_privada.key -out mensaje.firma mensaje.txt
   ```

4. Verifica la firma con la clave pública:

   ```bash
   openssl dgst -sha256 -verify clave_publica.pem -signature mensaje.firma mensaje.txt
   ```

   Debe responder `Verified OK`.

5. Comprueba que la firma detecta manipulaciones: verifica la firma contra el archivo modificado y observa que **falla**:

   ```bash
   openssl dgst -sha256 -verify clave_publica.pem -signature mensaje.firma mensaje_v2.txt
   ```

   Debe responder `Verification Failure`.

> **Evidencia 3:** captura donde se vean el `Verified OK` (archivo original) y el `Verification Failure` (archivo modificado).

---

### Parte B · Certificados digitales (PKI)

### Paso 5. Crear una Autoridad Certificadora (CA) propia

1. Genera la clave privada de la CA:

   ```bash
   openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:3072 -out ca.key
   ```

2. Crea el certificado autofirmado de la CA (válido 2 años). El comando pide datos; el estudiante completa país `CO`, y en *Common Name* escribe algo como `CA Curso Seguridad <NOMBRE>`:

   ```bash
   openssl req -x509 -new -nodes -key ca.key -sha256 -days 730 -out ca.crt
   ```

3. Revisa el contenido legible del certificado de la CA:

   ```bash
   openssl x509 -in ca.crt -noout -text | head -n 20
   ```

### Paso 6. Emitir un certificado para un sitio web

1. Genera la clave privada del servidor:

   ```bash
   openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out servidor.key
   ```

2. Crea una solicitud de firma de certificado (CSR). En *Common Name* escribe `localhost`:

   ```bash
   openssl req -new -key servidor.key -out servidor.csr
   ```

3. Crea un archivo de extensiones para que el certificado sea válido en navegadores modernos (obliga el campo *Subject Alternative Name*). El estudiante crea `servidor.ext` con este contenido (`nano servidor.ext`):

   ```
   authorityKeyIdentifier=keyid,issuer
   basicConstraints=CA:FALSE
   keyUsage=digitalSignature,keyEncipherment
   subjectAltName=@alt_names

   [alt_names]
   DNS.1=localhost
   IP.1=127.0.0.1
   ```

4. La CA firma la solicitud y emite el certificado del servidor (válido 1 año):

   ```bash
   openssl x509 -req -in servidor.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
     -out servidor.crt -days 365 -sha256 -extfile servidor.ext
   ```

5. Verifica que el certificado del servidor fue emitido correctamente por la CA:

   ```bash
   openssl verify -CAfile ca.crt servidor.crt
   ```

   Debe responder `servidor.crt: OK`.

> **Evidencia 4:** captura donde se vean el `servidor.crt: OK` y las primeras líneas del certificado del servidor (`openssl x509 -in servidor.crt -noout -text | head -n 15`).

### Paso 7. Probar el certificado en un servidor HTTPS local

1. Levanta un servidor HTTPS de prueba con el certificado emitido (OpenSSL trae uno incorporado):

   ```bash
   openssl s_server -accept 4443 -cert servidor.crt -key servidor.key -www
   ```

   La terminal queda ocupada atendiendo peticiones; no debe cerrarse.

2. En **otra terminal**, consulta el servidor confiando en la CA propia:

   ```bash
   cd ~/semana3-cripto
   echo | openssl s_client -connect localhost:4443 -CAfile ca.crt 2>/dev/null | grep -E "subject=|issuer=|Verify return code"
   ```

   Debe mostrar `Verify return code: 0 (ok)`.

3. Opcionalmente, abre en Firefox <https://localhost:4443>. El navegador mostrará una advertencia porque no conoce la CA propia; el estudiante explica en el informe **por qué** aparece esa advertencia y cómo se resolvería en un entorno real (instalando la CA o usando una CA pública como Let's Encrypt).

4. Detén el servidor con `Ctrl+C` en la primera terminal.

> **Evidencia 5:** captura del `Verify return code: 0 (ok)`.

---

### Parte C · Almacenamiento de contraseñas y control de acceso

### Paso 8. Analizar cómo se guardan las contraseñas

1. El estudiante recupera el código de Juice Shop de la semana 2 (o lo vuelve a clonar) y busca cómo se cifran las contraseñas:

   ```bash
   cd ~/semana2-sonarqube/juice-shop 2>/dev/null || (cd ~ && git clone --depth 1 https://github.com/juice-shop/juice-shop.git && cd juice-shop)
   grep -rn "createHash\|md5\|hashSync\|security.hash" lib/ models/ 2>/dev/null | head
   ```

2. En el informe explica:
   - ¿Qué algoritmo usa la aplicación para las contraseñas? (Juice Shop usa MD5 sin sal, de forma deliberada.)
   - ¿Por qué ese esquema es inseguro? (rapidez del cálculo, tablas rainbow, ausencia de sal.)
   - ¿Cuál sería el esquema correcto según OWASP? Debe citar la recomendación de **Argon2id** (o bcrypt como alternativa) de la guía de almacenamiento de contraseñas.

3. Demuestra el problema: calcula el MD5 de una contraseña común y comprueba lo fácil que es reconocerla:

   ```bash
   echo -n "admin123" | md5sum
   ```

   El estudiante explica que ese mismo hash es idéntico para cualquier usuario que use `admin123`, precisamente porque no hay sal.

### Paso 9. Relacionar con el control de acceso

En el informe, el estudiante identifica en la aplicación (o en la lectura de la semana) **un caso de control de acceso** y responde brevemente: ¿qué diferencia hay entre autenticación (verificar identidad) y autorización (verificar permisos)? ¿Qué principio de control de acceso aplicaría (mínimo privilegio, denegar por defecto)? Debe apoyarse en la lectura [A01 Pérdida de Control de Acceso](https://owasp.org/Top10/2021/es/A01_2021-Broken_Access_Control/).

### Paso 10. Limpiar y proteger las claves

1. El estudiante comprueba qué archivos generó:

   ```bash
   ls -l ~/semana3-cripto
   ```

2. **Importante:** las claves privadas (`clave_privada.key`, `ca.key`, `servidor.key`) **no se entregan ni se suben a ningún repositorio**. Para la entrega solo se incluyen las claves públicas, los certificados (`.crt`), la firma y las capturas. El estudiante explica en el informe por qué una clave privada nunca debe salir del equipo donde se generó.

---

## 5. Entregables

Un único archivo comprimido **`Semana3_Apellido_Nombre.zip`** que contenga:

1. **Informe** (`Informe_Semana3.pdf`, formato APA 7, 6 a 10 páginas de cuerpo) con:
   - Entorno de trabajo (versión de Kali y de OpenSSL).
   - Parte A: explicación de cada operación (cifrado, hash, firma) con las evidencias 1, 2 y 3.
   - Parte B: descripción de la CA y el certificado emitido, con las evidencias 4 y 5, y explicación de la advertencia del navegador.
   - Parte C: análisis del almacenamiento de contraseñas de Juice Shop, propuesta de corrección (Argon2id/bcrypt) y respuestas del Paso 9.
   - Reflexión final: ¿dónde encaja cada mecanismo criptográfico en una aplicación web segura?
   - Referencias en formato APA 7 (mínimo cinco, tres oficiales).
2. **Evidencias** `Evidencia1` a `Evidencia5` (capturas PNG) y las **claves públicas y certificados** generados (`clave_publica.pem`, `ca.crt`, `servidor.crt`). **No incluir claves privadas (`.key`).**

---

## 6. Rúbrica de evaluación (100 puntos)

| Criterio | Puntos | Excelente | Aceptable | Insuficiente |
|---|---|---|---|---|
| Cifrado simétrico y descifrado (Parte A) | 15 | Cifra y descifra correctamente, explica `-pbkdf2` y `-salt` (15) | Ejecuta los comandos sin explicar (9) | No lo logra (0) |
| Hash e integridad | 15 | Muestra el efecto avalancha y justifica por qué MD5 es inseguro (15) | Calcula hashes sin análisis (9) | No lo hace (0) |
| Par de claves y firma digital | 20 | Firma y verifica, y demuestra la detección de manipulación (`Verification Failure`) (20) | Firma pero no demuestra la manipulación (12) | No firma (0) |
| Certificados / PKI (Parte B) | 25 | CA y certificado emitidos, `verify OK`, servidor HTTPS probado y advertencia explicada (25) | Emite el certificado pero no lo prueba en el servidor (15) | No completa la PKI (0) |
| Contraseñas y control de acceso (Parte C) | 10 | Identifica MD5 sin sal, propone Argon2id/bcrypt y distingue autenticación de autorización (10) | Análisis parcial (6) | Sin análisis (0) |
| Norma APA 7 y calidad del informe | 15 | Formato, citas y referencias correctas; mínimo cinco, tres oficiales (15) | Errores menores o menos de cinco referencias (9) | Sin citas ni referencias (0) |

**Escala:** 90–100 sobresaliente · 70–89 aprobado · menor de 70 no aprobado.

---

## 7. Solución de problemas frecuentes

| Síntoma | Causa probable | Solución |
|---|---|---|
| `bad decrypt` al descifrar | Contraseña equivocada u opciones distintas a las del cifrado | Repetir usando exactamente las mismas opciones (`-aes-256-cbc -pbkdf2`) y la contraseña correcta |
| `Verification Failure` cuando debería verificar | El archivo cambió o se usó otra clave pública | Verificar la firma contra el archivo original y con la clave pública correspondiente a la privada usada al firmar |
| `unable to load Private Key` | Ruta o nombre de archivo equivocado | Comprobar con `ls` que el archivo existe en la carpeta actual |
| El navegador no confía en el certificado | Es normal: la CA es propia y no está en el almacén del navegador | Explicarlo en el informe; en producción se usaría una CA reconocida (por ejemplo Let's Encrypt) |
| `Connection refused` en el Paso 7.2 | El servidor `s_server` no está corriendo | Verificar que la primera terminal sigue con `s_server` activo |
| `Can't open ca.srl for reading` | Primera emisión con la CA | Usar `-CAcreateserial` (ya incluido en el comando del Paso 6.4) |

---

## 8. Recursos de apoyo

- [OWASP Top 10:2021 – A02 Fallas Criptográficas (español)](https://owasp.org/Top10/2021/es/A02_2021-Cryptographic_Failures/)
- [OWASP Top 10:2021 – A01 Pérdida de Control de Acceso (español)](https://owasp.org/Top10/2021/es/A01_2021-Broken_Access_Control/)
- [Criptografía para Ingenier@s – Jorge Ramió (libro gratuito, español)](https://www.criptored.es/paginas/Criptograf%C3%ADa%20para%20Ingenier@s.html)
- [INCIBE – Protege la información mediante técnicas criptográficas (español)](https://www.incibe.es/empresas/blog/protege-informacion-mediante-tecnicas-criptograficas)
- [OWASP Password Storage Cheat Sheet (inglés)](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- [OpenSSL – Documentación oficial (inglés)](https://docs.openssl.org/master/)
- [Kali Linux – Documentación oficial (inglés)](https://www.kali.org/docs/)

---

## 9. Advertencia ética

Las claves y certificados generados en esta actividad son para uso exclusivo en el laboratorio local. Emitir certificados que suplanten dominios o entidades reales, o usar estas técnicas sobre sistemas de terceros sin autorización escrita, constituye delito según la Ley 1273 de 2009.

---

*Material del módulo Desarrollo seguro, criptografía e IAM · Especialización en Seguridad Informática · Licencia CC BY-SA 4.0*
