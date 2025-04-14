---
title: "Credenciales"
classes: wide
header:
  teaser: /assets/images/site_data/c-credenciales.png
ribbon: teal
description: "Recomendaciones y requerimientos claves para aplicar buenas prácticas de código seguro en la gestión e implementación de credenciales"
categories:
  - codigo-seguro
tags:
  - credenciales
toc: true
---


# Introducción

Gestionar credenciales de forma segura es una de las bases del desarrollo de software responsable. No alcanza con tener contraseñas largas o agregar símbolos al azar: se trata de entender cómo se generan, almacenan, utilizan, rotan y eliminan dentro de un sistema, y de aplicar buenas prácticas desde el inicio.

En este post de la serie "Código Seguro", te comparto un recorrido por conceptos y recomendaciones que considero clave al momento de trabajar con credenciales. No solo vas a ver buenas prácticas, sino también ejemplos de malas implementaciones que —si no se corrigen a tiempo— pueden abrir la puerta a vulnerabilidades serias como acceso no autorizado, filtración de datos o escalamiento de privilegios.

Algunos de los puntos que abordo incluyen:

  - Cómo usar salts aleatorios y algoritmos de hash adecuados.
  - Cuál debería ser la longitud mínima de una contraseña y cómo construir una passphrase robusta.
  - Por qué es importante evitar la reutilización o el uso de contraseñas previamente filtradas.
  - Qué herramientas existen para gestionar credenciales (como Vault o Bitwarden).
  - Cuáles son los errores más comunes que conviene evitar.

  > ⚠️ Nota: los fragmentos de código fueron generados con ayuda de IA y se presentan solo con fines ilustrativos. No deben usarse directamente en producción sin revisión y adaptación al contexto real de tu proyecto.

Este post no apunta únicamente a cumplir normativas de seguridad. La idea es ir un paso más allá y pensar en credenciales como parte del diseño seguro del software desde el arranque.

# Contraseñas con salt aleatorio

Aplicar un `salt` aleatorio a cada contraseña antes de hashearla es una práctica fundamental para evitar ataques como `rainbow tables` o la correlación de hashes idénticos. El salt es un valor único, generado de forma impredecible, que se agrega a la contraseña antes de aplicar el algoritmo de hash. Esto garantiza que incluso si dos usuarios eligen la misma contraseña, los hashes resultantes sean completamente diferentes.

Un `salt` debe generarse de forma criptográficamente segura y ser distinto para cada usuario o entrada. Además, debe ser almacenado junto con el hash (pero no debe ser reutilizado ni predecible). Sin esta protección, incluso los algoritmos de hash más robustos pierden eficacia frente a ataques automatizados.

❌ Ejemplo de código inseguro

```python
import hashlib

def hash_password(password):
    # Inseguro: salt estático y predecible
    salt = "123456"
    return hashlib.sha256((salt + password).encode()).hexdigest()

hashed = hash_password("password123")
```

Este enfoque es inseguro porque:

  - Usa un salt fijo y compartido para todas las contraseñas.

  - Permite correlacionar hashes si los atacantes acceden a la base de datos.

  - No aprovecha la protección real que ofrece la aleatoriedad del salt.

✅ Ejemplo de código seguro

```python
import os
import hashlib

def hash_password(password):
    # Genera un salt aleatorio de 20 bytes
    salt = os.urandom(20)
    hashed = hashlib.pbkdf2_hmac('sha256', password.encode(), salt, 100000)
    return salt.hex() + ":" + hashed.hex()

hashed = hash_password("password123")
```

Este enfoque mejora significativamente la seguridad porque:

  - Usa un salt único y aleatorio para cada contraseña.

  - El salt se almacena junto al hash (por ejemplo, separado por :).

  - Aplica PBKDF2 con múltiples iteraciones, agregando resistencia a ataques de fuerza bruta.

💡 Recomendación práctica

Siempre generá un salt aleatorio por cada contraseña y almacenalo junto al hash resultante. Evitá reusar valores fijos o derivados del usuario (como el email o ID). Para sistemas modernos, preferí funciones como `bcrypt`, `scrypt` o `Argon2`, que ya incorporan salt y están diseñadas para ser lentas y seguras ante ataques automatizados. Recordá que el objetivo no es solo encriptar, sino también resistir ataques a gran escala.

# Almacenar contraseñas hasheadas

Una contraseña nunca debe ser almacenada en texto plano. En su lugar, debe ser transformada mediante un algoritmo de hash criptográfico que produzca un valor irreversible y único para cada entrada. A diferencia del cifrado, el hash no se puede “desencriptar”: es unidireccional, lo cual lo convierte en la opción adecuada para verificar contraseñas sin necesidad de conocerlas.

No todos los algoritmos de hash son adecuados. Funciones como `MD5` o `SHA-1` ya no son seguras, ya que existen colisiones conocidas y hardware capaz de realizar millones de cálculos por segundo. Para almacenar contraseñas de forma segura, se deben utilizar algoritmos diseñados específicamente para ese propósito, como `bcrypt`, `scrypt` o `Argon2`.

❌ Ejemplo de código inseguro

```python
import hashlib

def store_password(password):
    # Inseguro: uso de SHA-1 sin salt
    return hashlib.sha1(password.encode()).hexdigest()

hashed = store_password("SuperSecret2024!")
```
Este enfoque es inseguro porque:

  - SHA-1 es un algoritmo vulnerable a colisiones.

  - No utiliza salt, lo que permite correlacionar hashes entre usuarios.

  - Es demasiado rápido, lo que facilita ataques por fuerza bruta o diccionario.

✅ Ejemplo de código seguro

```python
import bcrypt

def store_password(password):
    # Seguro: hash usando bcrypt con salt incorporado
    hashed = bcrypt.hashpw(password.encode(), bcrypt.gensalt())
    return hashed.decode()

hashed = store_password("SuperSecret2024!")
```

Este enfoque es mucho más seguro porque:

  - Usa bcrypt, un algoritmo específicamente diseñado para almacenar contraseñas.

  - El salt es generado automáticamente e incluido en el hash final.

  - La función es lo suficientemente lenta para dificultar ataques masivos.

💡 Recomendación práctica

Usá funciones de hash pensadas para contraseñas, como `bcrypt`, `scrypt` o `Argon2`. Estas implementaciones ya incorporan mecanismos de salting, múltiples rondas de hashing y están diseñadas para ser costosas computacionalmente. Nunca almacenes contraseñas en texto plano ni uses funciones criptográficas generales como `SHA-256` sin protección adicional. La elección del algoritmo es crítica para resistir ataques modernos, incluso si la base de datos es comprometida.

# Contraseñas con al menos 20 caracteres

Una de las formas más simples y efectivas de mejorar la seguridad de una contraseña es aumentar su longitud. Mientras que muchos sistemas todavía permiten contraseñas de 8 u 11 caracteres, hoy se considera que una contraseña robusta debería tener al menos 20 caracteres, especialmente si se genera aleatoriamente.

La longitud es un factor crítico en la resistencia frente a ataques por fuerza bruta. Una contraseña corta puede ser adivinada rápidamente por herramientas automatizadas, incluso si contiene símbolos o mayúsculas. En cambio, una contraseña larga incrementa exponencialmente el espacio de búsqueda, haciéndola mucho más difícil de romper.

❌ Ejemplo de código inseguro

```python
# Inseguro: validación muy laxa
def is_valid_password(password):
    return len(password) >= 8

print(is_valid_password("Admin123"))  # débil
```
Este enfoque es riesgoso porque:

  - Acepta contraseñas muy cortas.

  - Depende únicamente de complejidad superficial (mayúsculas, números).

  - No resiste ataques por diccionario o fuerza bruta prolongada.

✅ Ejemplo de código seguro

```python
# Inseguro: validación muy laxa
# Seguro: exige al menos 20 caracteres
def is_valid_password(password):
    return len(password) >= 20

print(is_valid_password("5kG9#rZqA7mLp&cV2xEw"))  # Fuerte
```

Este enfoque mejora la seguridad porque:

  - Aumenta significativamente el esfuerzo necesario para forzar la contraseña.

  - Incentiva el uso de contraseñas generadas por herramientas, no recordadas manualmente.

  - Permite usar frases de paso o secuencias complejas sin depender únicamente de caracteres especiales.

💡 Recomendación práctica

Requiere contraseñas de al menos 20 caracteres en sistemas críticos o donde se utilicen claves maestras. Si usás generadores automáticos o gestores de contraseñas, asegurate de configurar una longitud mínima adecuada. En este caso, más es mejor.

# Validar contraseñas previamente utilizadas

Permitir que un usuario reutilice contraseñas anteriores debilita significativamente la seguridad del sistema. Esta práctica anula los beneficios del cambio de contraseña, especialmente si las anteriores ya fueron comprometidas o expuestas en bases de datos filtradas.

La validación de contraseñas previamente utilizadas consiste en almacenar un historial limitado de hashes antiguos y compararlos con la nueva contraseña antes de aceptarla. Esto no solo obliga a que cada nueva clave sea realmente nueva, sino que también fomenta el uso de contraseñas más seguras y menos predecibles.

❌ Ejemplo de código inseguro

```python
import bcrypt

# Simula la última contraseña usada (hash de "OldPassword123")
previous_hash = bcrypt.hashpw(b"OldPassword123", bcrypt.gensalt())

def is_password_new(password):
    # Inseguro: solo compara con la contraseña actual (hash no persistido)
    return not bcrypt.checkpw(password.encode(), previous_hash)

print(is_password_new("OldPassword123"))  # False
```

Este enfoque es inseguro porque:

  - Solo almacena un único hash anterior, sin mantener historial real.

  - No permite controlar cuántas versiones atrás puede retroceder un usuario.

  - Se rompe si el sistema no guarda correctamente los hashes antiguos.

✅ Ejemplo de código seguro

```python
import bcrypt

# Simulación de hashes de contraseñas anteriores
previous_passwords = [
    bcrypt.hashpw(b"OldPassword1", bcrypt.gensalt()),
    bcrypt.hashpw(b"OldPassword2", bcrypt.gensalt()),
    bcrypt.hashpw(b"OldPassword3", bcrypt.gensalt())
]

def is_password_reused(new_password, old_hashes):
    for old_hash in old_hashes:
        if bcrypt.checkpw(new_password.encode(), old_hash):
            return True
    return False

print(is_password_reused("OldPassword2", previous_passwords))  # True
```
Este enfoque es seguro porque:

  - Compara la nueva contraseña con un historial completo de hashes anteriores.

  - Evita la reutilización de contraseñas ya conocidas o previamente filtradas.

  - Permite extender o limitar la cantidad de contraseñas recordadas según política.

💡 Recomendación práctica

Implementá una política de historial de contraseñas que evite la reutilización de las últimas 5 a 10 claves. Guardá únicamente los hashes, no las contraseñas originales, y usá algoritmos fuertes como bcrypt o Argon2. Complementá esta medida validando también contra listas de contraseñas comprometidas públicas (por ejemplo, con la API de Have I Been Pwned). Esta práctica ayuda a romper con patrones predecibles y a mantener el sistema más robusto frente a ataques dirigidos.

# Prevenir el uso de contraseñas comprometidas

Una contraseña puede parecer segura por su longitud o complejidad, pero si ya ha sido filtrada en una brecha de datos, su uso representa un riesgo inmediato. La reutilización de contraseñas expuestas es una de las principales causas de accesos no autorizados, especialmente en ataques automatizados como el `credential stuffing`.

Prevenir el uso de contraseñas comprometidas implica verificar si una nueva contraseña ya ha aparecido en bases de datos filtradas. Para ello, existen servicios públicos y APIs como `Have I Been Pwned` que permiten realizar búsquedas anónimas de hashes para validar si una contraseña ha sido comprometida previamente.

❌ Ejemplo de código inseguro

```python
# Inseguro: no realiza ninguna validación contra contraseñas filtradas
def is_valid_password(password):
    return len(password) >= 12 and any(c.isdigit() for c in password)

print(is_valid_password("Welcome123"))  # True, pero muy común y probablemente filtrada
```

Este enfoque es insuficiente porque:

  - Se basa únicamente en reglas de complejidad.

  - No detecta contraseñas ya expuestas en filtraciones masivas.

  - Permite el uso de claves ampliamente conocidas por atacantes.

✅ Ejemplo de código seguro

```python
import hashlib
import requests

def is_password_pwned(password):
    sha1 = hashlib.sha1(password.encode()).hexdigest().upper()
    prefix, suffix = sha1[:5], sha1[5:]
    url = f"https://api.pwnedpasswords.com/range/{prefix}"
    response = requests.get(url)

    if response.status_code != 200:
        raise RuntimeError("Error al consultar Have I Been Pwned")

    return any(line.split(':')[0] == suffix for line in response.text.splitlines())

print(is_password_pwned("Password123"))  # True si fue filtrada
```
Este enfoque es seguro porque:

  - Usa el método k-Anonymity para consultar solo un fragmento del hash (no expone la contraseña completa).

  - Valida si la contraseña fue encontrada en millones de registros filtrados.

  - Bloquea el uso de claves previamente comprometidas, aunque parezcan fuertes.

💡 Recomendación práctica

Integrá una validación contra listas de contraseñas filtradas al momento de registrar o cambiar una clave. Utilizá APIs confiables como Have I Been Pwned, que ofrecen consultas eficientes sin comprometer la privacidad del usuario. Esta medida es especialmente importante para accesos privilegiados o claves maestras. Recordá: una contraseña segura no es solo la que parece fuerte, sino la que nunca ha sido expuesta.

# Limitar la vida útil de las contraseñas

Toda contraseña, por más robusta que sea, se vuelve progresivamente vulnerable con el paso del tiempo. Cuanto más tiempo permanece en uso, más oportunidades existen para que sea interceptada, reutilizada en otros servicios, filtrada en una brecha de datos o comprometida por malware. Por eso, establecer una vida útil máxima para las contraseñas es una práctica esencial en sistemas que manejan información sensible.

Limitar la duración de una contraseña obliga al usuario a renovarla periódicamente, lo cual reduce el riesgo de exposición prolongada. Sin embargo, esta política debe implementarse con criterios razonables y combinada con otras medidas (como evitar la reutilización o verificar contraseñas comprometidas) para no generar fricción innecesaria ni incentivar malas prácticas, como el uso de patrones predecibles.

❌ Ejemplo de código inseguro

```python
# Inseguro: sin control de expiración
user = {
    "password_hash": "abc123...",
    "password_last_changed": None
}

def is_password_expired(user):
    return False  # Nunca expira
```

Este enfoque es inseguro porque:

  - Permite que una contraseña se mantenga activa indefinidamente.

  - No detecta cuándo fue cambiada por última vez.

  - Impide aplicar políticas de rotación periódica.

✅ Ejemplo de código seguro

```python
from datetime import datetime, timedelta

user = {
    "password_hash": "abc123...",
    "password_last_changed": datetime(2024, 12, 1)
}

def is_password_expired(user, max_age_days=90):
    return datetime.now() > user["password_last_changed"] + timedelta(days=max_age_days)

print(is_password_expired(user))  # True si pasaron más de 90 días
```

Este enfoque es más seguro porque:

  - Define una vida útil máxima para cada contraseña.

  - Permite forzar el cambio periódico en función de la fecha de último cambio.

  - Se adapta fácilmente a diferentes políticas según tipo de cuenta (admin, usuario común, etc.).

💡 Recomendación práctica

Establecé un período de expiración razonable para las contraseñas, por ejemplo, cada 90 o 180 días, dependiendo del nivel de criticidad del sistema. Evitá aplicar esta política en aislamiento: combiná el vencimiento con controles que impidan la reutilización, que validen si la nueva contraseña fue comprometida y que ofrezcan mecanismos de recuperación robustos. Recordá que el objetivo no es forzar cambios frecuentes, sino reducir el tiempo de exposición de una contraseña en caso de que sea vulnerada.

# Establecer un mecanismo de regeneración de contraseñas 

Toda aplicación que gestione usuarios debe contar con un mecanismo seguro y eficiente para permitir la regeneración de contraseñas, ya sea por olvido, expiración o requerimiento de seguridad (por ejemplo, ante un incidente de seguridad o sospecha de compromiso).

Un buen sistema de regeneración debe verificar adecuadamente la identidad del usuario, generar enlaces temporales seguros (tokens de un solo uso), establecer vencimientos breves, y evitar la exposición de datos sensibles. Además, debe garantizar que el nuevo acceso invalide inmediatamente la contraseña anterior o los tokens activos asociados.

❌ Ejemplo de código inseguro

```python
def generate_reset_link(user_email):
    # Inseguro: link de restablecimiento sin token, solo con el email
    return f"https://midominio.com/reset?email={user_email}"
```
Este enfoque es altamente inseguro porque:

  - No incluye ningún identificador único o secreto.

  - Permite que cualquier persona que conozca el correo genere un enlace válido.

  - Facilita ataques automatizados y suplantación de identidad.

✅ Ejemplo de código seguro

```python
import uuid
from datetime import datetime, timedelta

# Almacena el token y su expiración (ejemplo simulado)
reset_tokens = {}

def generate_reset_token(user_id):
    token = str(uuid.uuid4())
    reset_tokens[token] = {
        "user_id": user_id,
        "expires_at": datetime.now() + timedelta(minutes=15)
    }
    return f"https://midominio.com/reset-password?token={token}"

# Ejemplo de uso
print(generate_reset_token("user_123"))
```
Este enfoque es seguro porque:

  - Genera un token aleatorio, único e impredecible.

  - Establece una expiración estricta del enlace (por ejemplo, 15 minutos).

  - El token es de un solo uso y se valida antes de permitir la regeneración.

💡 Recomendación práctica

Implementá un mecanismo de restablecimiento de contraseñas basado en tokens temporales y de un solo uso, con vencimientos breves y validaciones estrictas. Evitá enviar contraseñas por correo electrónico y asegurate de registrar los intentos de regeneración para detectar abusos. Siempre que se regenere una contraseña, invalidá las sesiones activas y notificá al usuario del cambio.

# Definir una herramienta de gestión de contraseñas

Las `credenciales` de acceso, especialmente aquellas asociadas a usuarios con permisos elevados, deben ser administradas a través de herramientas especializadas. Usar métodos manuales o almacenar contraseñas en archivos de configuración, planillas o código fuente representa una amenaza directa a la seguridad de cualquier sistema.

Las herramientas de gestión de contraseñas como `HashiCorp Vault`, `Bitwarden`, `1Password CLI`, `KeePassXC` o `LastPass` permiten centralizar y controlar el ciclo de vida completo de las contraseñas: desde su creación hasta su rotación, expiración o eliminación. Estas herramientas ofrecen cifrado fuerte, registro de accesos, control granular por roles, y compatibilidad con integraciones automatizadas (por ejemplo, en pipelines de `CI/CD`).

❌ Ejemplo de código inseguro

```python
# Inseguro: contraseña escrita en texto plano dentro del código
SMTP_PASSWORD = "SuperSecret2024!"
```

✅ Ejemplo de código seguro

```python
import hvac  # Cliente de HashiCorp Vault en Python

client = hvac.Client(url='https://vault.miempresa.com', token='s.abc123token')

# Accede al secreto de forma segura
secret = client.secrets.kv.read_secret_version(path='servicios/email')
smtp_password = secret['data']['data']['password']
```

Con este enfoque:

  - La contraseña no está presente en el código.

  - Se puede revocar o rotar sin modificar la aplicación.

  - Vault registra cada acceso, incluso diferenciando por usuario o servicio.

  - Se puede restringir el acceso únicamente a los servicios o usuarios autorizados.

💡 Recomendación práctica

Seleccioná una herramienta que se ajuste al tamaño y complejidad de tu entorno. Para proyectos pequeños, una opción local como `KeePassXC` puede ser suficiente. En entornos corporativos o escalables, herramientas como `Vault` o `1Password Business` ofrecen mayores garantías de seguridad y trazabilidad. Lo importante es no delegar la gestión de credenciales a soluciones improvisadas o manuales.

# Forzar reautenticación

La autenticación inicial de un usuario no debería otorgar acceso ilimitado e indefinido a todas las funcionalidades críticas del sistema. En operaciones especialmente sensibles como cambiar una contraseña, modificar una clave API, acceder a información financiera o eliminar cuentas es una buena práctica forzar al usuario a reautenticarse, incluso si su sesión está activa.

Este mecanismo, conocido como reauthentication, obliga al usuario a ingresar nuevamente su contraseña o realizar una validación de segundo factor antes de permitir una acción crítica. De esta forma se reduce el riesgo asociado a sesiones comprometidas, accesos prolongados en dispositivos desatendidos, o ataques internos desde cuentas legítimas pero descuidadas.

❌ Ejemplo de código inseguro

```python
# Inseguro: permite cambiar la contraseña sin validar la contraseña actual
def change_password(user_id, new_password):
    # Falta validación del usuario autenticado
    update_password(user_id, new_password)
```

Este enfoque es riesgoso porque:

  - Confía ciegamente en la sesión activa.

  - Permite que un atacante con acceso al navegador o a una sesión comprometida realice cambios críticos.

  - No verifica si quien realiza la operación es realmente el propietario de la cuenta.

✅ Ejemplo de código seguro

```python
# Simula un paso de reautenticación antes de permitir el cambio
def reauthenticate(user_id, current_password):
    stored_hash = get_password_hash_from_db(user_id)
    return verify_password(current_password, stored_hash)

def change_password(user_id, current_password, new_password):
    if not reauthenticate(user_id, current_password):
        raise Exception("Reautenticación fallida")
    update_password(user_id, new_password)
```

Este enfoque es más seguro porque:

  - Solicita la contraseña actual antes de permitir cambios.

  - Requiere validación explícita del usuario en operaciones sensibles.

  - Puede extenderse fácilmente para incorporar MFA como segundo paso.

💡 Recomendación práctica

Identificá claramente las acciones de alto impacto dentro de tu aplicación (por ejemplo, cambiar correo electrónico, credenciales, deshabilitar MFA, eliminar datos sensibles) y exigí reauthenticación inmediata antes de permitirlas. Este mecanismo debe ser claro, rápido, y seguro, sin exponer contraseñas en texto plano ni generar tokens persistentes. Si tu sistema utiliza MFA, considerá forzar también una segunda validación. Recordá que la sesión activa no siempre equivale a consentimiento o control activo del usuario.

# ✅ Checklist de gestión segura de credenciales

- [x] Uso de funciones de hash robustas como `bcrypt`, `scrypt` o `Argon2`.
- [x] Generación de `salt` aleatorio, único e impredecible por contraseña.
- [x] Prohibición de algoritmos obsoletos como `MD5` o `SHA-1`.
- [x] Contraseñas de al menos 20 caracteres o uso de passphrases.
- [x] Validación contra listas públicas de contraseñas comprometidas.
- [x] Prevención de la reutilización de contraseñas recientes.
- [x] Definición de una vida útil limitada para las contraseñas.
- [x] Regeneración segura con tokens temporales y únicos.
- [x] Invalidación de sesiones tras cambio de contraseña.
- [x] Reautenticación en operaciones críticas.
- [x] Aplicación de MFA para accesos privilegiados.
- [x] Uso de herramientas como `Vault`, `Bitwarden`, etc.
- [x] Eliminación de secretos hardcodeados.
- [x] Control de acceso a credenciales por rol o entorno.
- [x] Registro de eventos críticos de autenticación.
- [x] Auditoría y monitoreo periódico de actividad.

> Este checklist resume prácticas clave. Su implementación ayuda a reducir riesgos comunes en el manejo de credenciales.

# 📚 Referencias técnicas

## 🧪 Fluid Attacks

- [Credential Management: Security Criteria](https://help.fluidattacks.com/portal/en/kb/criteria/requirements/credentials)
- [Insecure Storage of Credentials](https://help.fluidattacks.com/portal/en/kb/articles/criteria-vulnerabilities-085)
- [Weak Authentication Mechanisms](https://help.fluidattacks.com/portal/en/kb/articles/criteria-vulnerabilities-363)
- [Password Policy Misconfigurations](https://help.fluidattacks.com/portal/en/kb/articles/criteria-fixes-cloudformation-363)

## 🧱 CWE (Common Weakness Enumeration)

- [CWE-256: Plaintext Storage of a Password](https://cwe.mitre.org/data/definitions/256.html)
- [CWE-259: Use of Hard-coded Password](https://cwe.mitre.org/data/definitions/259.html)
- [CWE-521: Weak Password Requirements](https://cwe.mitre.org/data/definitions/521.html)
- [CWE-916: Use of Password Hash With Insufficient Computational Effort](https://cwe.mitre.org/data/definitions/916.html)
- [CWE-798: Use of Hard-coded Credentials](https://cwe.mitre.org/data/definitions/798.html)

## 🔐 OWASP (Open Worldwide Application Security Project)

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- [OWASP Top 10 - A07:2021 – Identification and Authentication Failures](https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/)
- [OWASP ASVS – V2: Authentication Verification Requirements](https://github.com/OWASP/ASVS)
- [OWASP Cryptographic Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html)

## 🧩 Sonar (SonarQube / SonarCloud Rules)

- [S2068: Hardcoded credentials (Sonar rule)](https://rules.sonarsource.com/java/RSPEC-2068/)
- [S5131: Use of Weak Cryptographic Algorithms](https://rules.sonarsource.com/java/RSPEC-5131/)
- [S1313: Hardcoded IP addresses and sensitive data](https://rules.sonarsource.com/java/RSPEC-1313/)
- [S6333: Improper Credential Storage](https://rules.sonarsource.com/javascript/RSPEC-6333/)
- [Security Hotspots – Credentials Management](https://docs.sonarsource.com/sonarqube/latest/user-guide/security-hotspots/)
