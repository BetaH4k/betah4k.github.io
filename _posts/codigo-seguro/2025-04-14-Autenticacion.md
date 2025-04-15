---
title: "Autenticación"
classes: wide
header:
  teaser: /assets/images/site_data/c-autenticacion.png
ribbon: teal
description: "Recomendaciones y requerimientos claves para aplicar buenas prácticas de código seguro en la gestión e implementación de autenticación en una aplicación."
categories:
  - Codigo Seguro
tags:
  - autenticacion
toc: true
---


# Introducción

Gestionar la autenticación de forma segura es una de las bases del desarrollo de software responsable. No basta con implementar formularios de login o almacenar contraseñas correctamente: se trata de comprender cómo se validan, gestionan y protegen los accesos en cada punto del proceso, y de aplicar buenas prácticas desde el inicio.

En este post de la serie "Código Seguro", te comparto un recorrido por conceptos y recomendaciones que considero clave al momento de trabajar con la autenticación. No solo vas a encontrar buenas prácticas, sino también ejemplos de implementaciones deficientes que —si no se corrigen a tiempo— pueden abrir la puerta a vulnerabilidades serias como accesos no autorizados, suplantación de identidad o ataques de fuerza bruta.

Algunos de los puntos que abordo incluyen:

  - Cómo utilizar protocolos de autenticación estandarizados y comprobados.
  - La importancia de implementar autenticación multifactor (MFA) en sistemas críticos.
  - Estrategias para evitar ataques de enumeración y asegurar tiempos de respuesta uniformes.
  - Prácticas para gestionar sesiones y renovar tokens de forma segura.
  - Los errores más comunes en autenticación y cómo prevenirlos.
  
  > ⚠️ Nota: los fragmentos de código fueron generados con ayuda de IA y se presentan solo con fines ilustrativos. No deben usarse directamente en producción sin revisión y adaptación al contexto real de tu proyecto.

Este post no apunta únicamente a cumplir normativas y requisitos de seguridad. La idea es ir un paso más allá y pensar en la autenticación como parte del diseño seguro del software desde el comienzo.

# Evitar el bloqueo de cuentas por intentos fallidos

Una práctica común para proteger cuentas contra ataques es bloquear el acceso después de varios intentos fallidos de autenticación. Sin embargo, si no se implementa correctamente, esta medida puede facilitar ataques de denegación de servicio (DoS), en los que un atacante bloquea cuentas legítimas deliberadamente.

Los bloqueos de cuenta son especialmente problemáticos en servicios críticos o sistemas donde la recuperación del acceso es compleja. En vez de bloquear por completo, una alternativa más robusta es aplicar retrasos progresivos (rate limiting) o requerir CAPTCHA tras múltiples intentos fallidos.

❌ Ejemplo de código inseguro

```python
# Inseguro: bloqueo permanente tras 5 intentos
if failed_attempts >= 5:
    lock_account(user_id)
```

Este enfoque puede ser explotado fácilmente:

  - Permite que un atacante bloquee cuentas ajenas.
  - No diferencia entre ataques automatizados y errores genuinos del usuario.
  - No considera la procedencia o el contexto del intento.

✅ Ejemplo de código seguro

```python
# Seguro: aplica retrasos crecientes según número de fallos
def delay_login(failed_attempts):
    delay = min(failed_attempts * 2, 30)  # máximo 30 segundos
    time.sleep(delay)
```

Este enfoque mejora significativamente la seguridad porque:

  - Usa un salt único y aleatorio para cada contraseña.
  - El salt se almacena junto al hash (por ejemplo, separado por :).
  - Aplica PBKDF2 con múltiples iteraciones, agregando resistencia a ataques de fuerza bruta.

💡 Recomendación práctica

Evitá bloquear cuentas automáticamente. En su lugar, utilizá retrasos exponenciales, CAPTCHA, o validaciones secundarias para responder a múltiples intentos fallidos. Reservá el bloqueo total solo para casos extremos y asegurate de tener mecanismos de recuperación efectivos y seguros.

# Solicitud segura de autenticación y credenciales

El proceso de autenticación comienza con una solicitud explícita para acceder al sistema o a un recurso protegido. Este punto de entrada es crítico, ya que es donde el sistema debe decidir cuándo y cómo permitir que un usuario inicie el proceso de autenticación, ya sea proporcionando credenciales manualmente o a través de un flujo automatizado (como tokens de acceso o SSO).

Un mal diseño en esta etapa puede permitir ataques como enumeración de usuarios, abuso de endpoints no protegidos o una interfaz de autenticación que revele demasiado sobre el sistema interno. La solicitud de autenticación debe ser clara, segura, y no revelar si un usuario existe o no en el sistema.

❌ Ejemplo de código inseguro

```python
# Inseguro: expone si el usuario existe
def request_authentication(username):
    if not user_exists(username):
        return "Usuario no encontrado"
    return "Ingrese su contraseña"
```
Este enfoque tiene varios riesgos:

  - Permite a un atacante validar qué usuarios están registrados.
  - Facilita ataques de enumeración mediante diccionarios comunes.
  - Rompe el principio de respuesta uniforme (indistinguible entre éxito y error).

✅ Ejemplo de código seguro

```python
# Seguro: respuesta uniforme, sin revelar existencia del usuario
def request_authentication(username):
    # Lógica interna para registrar el intento, pero sin revelar estado
    log_attempt(username)
    return "Si las credenciales son correctas, podrá continuar"
```

Este enfoque mejora la seguridad porque:

  - Ofrece una respuesta genérica para todos los casos.
  - No revela si el usuario existe o no.
  - Protege la privacidad y la integridad del sistema desde el primer contacto.

💡 Recomendación práctica

Diseñar los endpoints de solicitud de autenticación de forma tal que siempre respondan con mensajes genéricos, independientemente de si el usuario existe o no. No brindar detalles innecesarios sobre el estado de la cuenta. 

# Validar credenciales con protocolos estándar

Cuando una aplicación recibe una contraseña, un token o cualquier tipo de credencial, necesita validar que esa información es auténtica y que la persona que intenta acceder tiene derecho a hacerlo. Pero no alcanza con verificar que la credencial "coincida" con lo que está en la base de datos: también es clave cómo se hace esa verificación.

Una validación mal implementada puede derivar en accesos no autorizados, filtraciones o ataques por tiempo de respuesta. Por eso, conviene apoyarse en estándares abiertos y ampliamente auditados como OAuth 2.0, OpenID Connect, SAML o funciones de hashing diseñadas específicamente para contraseñas, como bcrypt, scrypt o Argon2.

También es importante establecer una interfaz clara para recibir y procesar credenciales, especialmente en sistemas que deben soportar múltiples métodos de autenticación (contraseña, MFA, token, biometría, etc.). Todo eso debe estar definido de forma coherente, validando correctamente cada tipo de dato y evitando comparaciones inseguras.

❌ Ejemplo de código inseguro

```python
# Inseguro: comparación directa sin hash ni controles
def authenticate(username, password):
    stored = database.get_user_password(username)
    return password == stored
```

Este enfoque es riesgoso porque:

  - Compara contraseñas en texto plano.
  - No aplica ningún mecanismo de protección ante ataques por tiempo de respuesta.
  - No implementa funciones de hash ni protocolos estándar.
  - Permite explotar cualquier fuga previa de contraseñas en la base.

✅ Ejemplo de código seguro

```python
# Seguro: uso de bcrypt para validar la credencial
import bcrypt

def validate_credentials(username, password, db_session):
    user = db_session.get_user(username)
    if not user:
        return False
    return bcrypt.checkpw(password.encode(), user.hashed_password)
```
Este enfoque mejora significativamente la seguridad porque:

  - Usa hashing robusto con salt incorporado.
  - No compara directamente contraseñas en texto plano.
  - Resiste ataques por fuerza bruta y filtraciones parciales.
  - Puede extenderse fácilmente para incluir MFA u otros factores.

💡 Recomendación práctica

Siempre validar credenciales utilizando funciones o protocolos ampliamente reconocidos y probados. Evitar implementar soluciones caseras para comparar contraseñas o manejar tokens. Usar `bcrypt`, `scrypt` o `Argon2` para contraseñas, y `OAuth/OpenID` para autenticación federada. Definir una interfaz clara entre los distintos módulos que intervienen en la autenticación para que sea más fácil auditar, extender y proteger.

# Autenticación multifactor y tokens seguros

Confiar únicamente en una contraseña para proteger una cuenta ya no es suficiente. La mayoría de las filtraciones y accesos no autorizados que ocurren hoy en día involucran credenciales robadas o reutilizadas. Por eso, agregar un segundo factor de autenticación (MFA) se volvió una medida esencial para proteger accesos críticos.

El MFA combina al menos dos elementos distintos:

  - Algo que sabés (una contraseña).
  - Algo que tenés (una app de autenticación o token físico).
  - Algo que sos (biometría).

Implementarlo correctamente no solo implica exigir un segundo paso, sino también:

  - Evitar mecanismos débiles como las preguntas de seguridad (conocidas como KBA).
  - Controlar la duración de los tokens fuera de banda.
  - Asegurar que cada cuenta tenga sus propios factores de autenticación.

❌ Ejemplo de implementación insegura

```python
# Inseguro: pregunta de seguridad como segundo factor
def verify_second_factor(answer):
    return answer.lower() == "boca juniors"
```

Este enfoque es problemático porque:

  - Las respuestas suelen ser fáciles de adivinar o buscar en redes sociales.
  - No ofrece protección real frente a ataques automatizados.
  - Genera una falsa sensación de seguridad.

✅ Ejemplo de implementación segura

```python
# Seguro: uso de TOTP con tiempo limitado
import pyotp

def verify_totp(user_secret, token):
    totp = pyotp.TOTP(user_secret)
    return totp.verify(token)
```

Este enfoque es mucho más robusto porque:

  - Utiliza un algoritmo de clave temporal (TOTP) sincronizado con el tiempo del sistema.
  - Los tokens duran solo 30 segundos y son de un solo uso.
  - Cada cuenta tiene su secreto único, lo que impide el reuso de tokens entre usuarios.

💡 Recomendación práctica

Aplicar MFA en todos los accesos críticos del sistema. Evitar preguntas de seguridad o validaciones triviales. Implementar métodos como TOTP (por ejemplo, con Google Authenticator o Authy), claves FIDO2 o tokens físicos. Limitar la duración de los tokens fuera de banda a unos pocos minutos y asegurarse de que cada cuenta tenga su configuración de MFA independiente. Si un factor se pierde o expira, ofrecer un mecanismo de recuperación seguro pero riguroso, como la reautenticación con revisión manual o por correo verificado.

# Respuestas seguras ante intentos de autenticación

Cada vez que alguien intenta autenticarse en una aplicación, el sistema devuelve una respuesta: acceso concedido, acceso denegado, error de contraseña, usuario no encontrado, etc. Estas respuestas, aunque parezcan inofensivas, pueden convertirse en una fuente de información para atacantes si no están bien diseñadas.

Mensajes muy específicos —como “usuario inexistente” o “contraseña incorrecta”— permiten ataques de enumeración de usuarios, mientras que los bloqueos automáticos por múltiples fallos pueden ser usados para lanzar ataques de denegación de servicio contra cuentas legítimas. Además, tiempos de respuesta diferentes entre una autenticación exitosa y una fallida pueden revelar detalles internos del sistema.

Lo ideal es que las respuestas del sistema:

  - Sean genéricas y no revelen si el usuario existe.
  - Tengan tiempos de respuesta constantes.
  - No bloqueen cuentas automáticamente tras pocos errores.

❌ Ejemplo de código inseguro

```python
# Inseguro: revela si el usuario existe
def login(username, password):
    if not user_exists(username):
        return "El usuario no existe"
    if not validate_password(username, password):
        return "Contraseña incorrecta"
    return "Bienvenido"
```

Este enfoque tiene varios problemas:

  - Permite verificar si un usuario está registrado.
  - Entrega información valiosa a un atacante sin necesidad de probar contraseñas.
  - Tiene tiempos de ejecución diferentes entre cada rama del flujo.

✅ Ejemplo de implementación segura

```python
import time

# Simula un tiempo de respuesta uniforme y mensajes genéricos
def login(username, password):
    time.sleep(1.5)  # Tiempo fijo para evitar análisis de timing
    user = get_user(username)
    if not user or not validate_password(user, password):
        return "Las credenciales ingresadas no son válidas"
    return "Acceso concedido"
```

Este enfoque es más seguro porque:

  - No revela si el problema está en el usuario o en la contraseña.
  - Mantiene el mismo tiempo de respuesta en todos los casos.
  - Previene ataques de enumeración y análisis por canal lateral (timing attacks).

💡 Recomendación práctica

Diseñar las respuestas del sistema de manera que no revelen información sensible. Usar siempre mensajes genéricos para errores de autenticación, aplicar tiempos de espera constantes y evitar bloquear cuentas automáticamente tras múltiples fallos. Si se decide bloquear temporalmente, que sea mediante mecanismos como delays progresivos o reCAPTCHA, y no con bloqueos definitivos. El objetivo es proteger sin castigar al usuario legítimo por errores genuinos.

# Verificación adicional: biometría y prueba de humanidad

En ciertos contextos, especialmente en operaciones sensibles o accesos con privilegios elevados, una contraseña o token no es suficiente. Para reforzar la seguridad y reducir el riesgo de automatización maliciosa, muchas aplicaciones suman una capa extra de verificación que puede tomar dos formas:

  - Biometría: como huellas digitales, reconocimiento facial o escaneo de iris.
  - Pruebas de humanidad: como CAPTCHAs o desafíos interactivos que validan que quien está al otro lado es una persona real.

Estas medidas buscan mitigar distintos riesgos. La biometría agrega un factor que no se puede perder ni compartir tan fácilmente (aunque tiene otros desafíos), mientras que los tests de humanidad protegen contra bots, scripts automatizados o ataques por fuerza bruta que prueban miles de credenciales por minuto.

❌ Ejemplo de verificación insegura

```python
# Inseguro: permite continuar sin verificar interacción humana
def submit_login(username, password):
    return authenticate(username, password)
```

Este flujo es vulnerable porque:

  - No incorpora ninguna barrera contra bots automatizados.
  - Puede ser explotado en ataques de fuerza bruta o credential stuffing.
  - No tiene controles contextuales (dispositivo, localización, velocidad de interacción).

✅ Ejemplo de implementación segura

```python
# Seguro: requiere validación humana antes de procesar el login
def submit_login(username, password, captcha_token):
    if not validate_captcha(captcha_token):
        return "Por favor completá la verificación"
    return authenticate(username, password)
```
Este enfoque es más robusto porque:

  - Integra una verificación activa de humanidad antes del intento de login.
  - Reduce significativamente la eficacia de ataques automatizados.
  - Puede adaptarse fácilmente a distintos mecanismos (reCAPTCHA, hCaptcha, etc.).

💡 Recomendación práctica

Agregar mecanismos de verificación adicional en puntos clave del sistema. Para accesos críticos o repetidos, considerar incorporar factores biométricos o verificaciones de actividad humana (CAPTCHA, patrones de interacción, análisis de comportamiento). Estas medidas no deben reemplazar la autenticación tradicional, sino complementarla.

# Supervisar el acceso: notificaciones y verificación del dispositivo

La seguridad no termina cuando una persona se loguea. De hecho, muchas de las acciones más importantes en un sistema seguro ocurren después de la autenticación: registrar quién accedió, desde qué dispositivo, y notificar al usuario si algo fuera de lo normal sucede.

Por eso, es fundamental implementar medidas que permitan detectar accesos sospechosos, alertar al usuario en tiempo real y, cuando sea necesario, verificar la identidad del dispositivo desde el cual se realiza la conexión.

Esto no solo mejora la seguridad general del sistema, sino que también empodera al usuario: si recibe una notificación de acceso desde un país o navegador que no reconoce, puede actuar rápidamente para proteger su cuenta.

❌ Ejemplo de implementación sin trazabilidad

```python
# Inseguro: no registra ni notifica nada
def login(username, password):
    if authenticate(username, password):
        return "Bienvenido"
```

Este enfoque es débil porque:

  - No registra información sobre el acceso (ubicación, IP, dispositivo).
  - No notifica al usuario sobre actividad en su cuenta.
  - Hace que detectar accesos no autorizados sea casi imposible.

✅ Ejemplo de implementación segura

```python
# Seguro: registra y notifica al usuario tras un acceso exitoso
def login(username, password, device_info):
    if not authenticate(username, password):
        return "Credenciales inválidas"
    
    log_access(username, device_info)
    send_access_notification(username, device_info)
    return "Acceso concedido"
```

Este enfoque fortalece la seguridad porque:

  - Registra cada acceso con datos relevantes (IP, agente, sistema operativo, hora).
  - Notifica al usuario inmediatamente, dándole visibilidad.
  - Permite identificar comportamientos anómalos o dispositivos desconocidos.

💡 Recomendación práctica

Implementar mecanismos de registro de accesos con detalles como dirección IP, ubicación estimada, navegador y sistema operativo. Además, enviar una notificación automática al usuario cada vez que se accede a su cuenta desde un nuevo dispositivo o ubicación. Para reforzar esta medida, considerar incluir controles como listas de dispositivos autorizados, verificación de identidad del equipo mediante certificados o tokens, y la posibilidad de revocar sesiones activas desde el panel de usuario.

# Control del ciclo de autenticación: expiración y recuperación

Una autenticación no debería durar para siempre. Tanto las sesiones activas como los tokens de acceso tienen que tener una vida útil limitada, especialmente en sistemas sensibles o expuestos a internet. Permitir que una sesión siga válida indefinidamente, aunque el usuario cierre el navegador o deje de interactuar, es una invitación a accesos indebidos.

Por otro lado, también hay que prever qué pasa si el usuario pierde acceso a su cuenta, ya sea porque olvidó la contraseña, cambió de dispositivo o fue víctima de un ataque. El sistema debe ofrecer un mecanismo de recuperación robusto, seguro y con controles adicionales, sin dejar puertas abiertas al abuso o a la suplantación de identidad.

❌ Ejemplo de sesión sin expiración y recuperación insegura

```python
# Inseguro: la sesión no expira y el enlace de recuperación es débil
SESSION_DURATION = None  # Sesión permanente

def generate_reset_link(user_email):
    return f"https://miapp.com/reset?email={user_email}"
```

Este diseño presenta varios riesgos:

  - Las sesiones activas podrían seguir funcionando por semanas o meses.
  - Cualquier persona que intercepte el link de recuperación puede acceder a la cuenta.
  - No hay validación temporal ni verificación adicional.

✅ Ejemplo de control de sesión y recuperación segura

```python
from datetime import datetime, timedelta
import uuid

# Control de expiración
SESSION_DURATION = timedelta(minutes=30)

# Generación de token temporal
def generate_secure_reset_token(user_id):
    token = str(uuid.uuid4())
    store_token(user_id, token, expires_at=datetime.now() + timedelta(minutes=15))
    return f"https://miapp.com/reset-password?token={token}"
```

Este enfoque es mucho más seguro porque:

  - Limita la duración de la sesión o autenticación a un tiempo razonable.
  - Usa un token único, aleatorio y con expiración para el restablecimiento.
  - Permite invalidar tokens una vez utilizados o si se detecta un uso anómalo.

💡 Recomendación práctica

Establecer límites temporales para sesiones, tokens de acceso y autenticaciones prolongadas. Usar cookies seguras, expiraciones claras y mecanismos de renovación controlados. Para recuperación de cuentas, generar tokens únicos y temporales, con vencimientos breves y una sola oportunidad de uso. Siempre que sea posible, agregar una segunda capa de validación (por ejemplo, un código enviado al correo o teléfono registrado). No confiar únicamente en el email como prueba de identidad.

# Opciones de autenticación seguras y equitativas

En muchas aplicaciones se ofrecen distintos métodos para autenticarse: contraseña tradicional, autenticación vía redes sociales, login con cuenta corporativa, clave temporal por email, entre otros. Esta flexibilidad puede ser útil para mejorar la experiencia del usuario, pero también representa un riesgo si no se cuida la seguridad de cada uno de esos caminos.

No sirve de nada tener una autenticación robusta por contraseña y MFA si también se permite ingresar solo con un enlace por email sin verificación extra. Todos los métodos habilitados deben cumplir un estándar de seguridad equivalente, o al menos proporcional al nivel de acceso que otorgan.

Además, es importante que ningún método alternativo funcione como un "atajo" inseguro que los atacantes puedan explotar. Si una opción es más débil, será la primera que se intente vulnerar.

❌ Ejemplo de implementación desigual

```python
# Inseguro: acceso completo con solo un token por correo
def login_with_email_token(token):
    user = get_user_from_token(token)
    if user:
        return "Acceso total"
```

Este enfoque es inseguro porque:

  - El token puede ser interceptado o reutilizado si no tiene vencimiento o protección.
  - No exige un segundo factor ni verificación adicional.
  - Otorga acceso completo sin requerir autenticación robusta.

✅ Ejemplo de diseño equitativo

```python
# Seguro: requiere doble validación incluso en flujos alternativos
def login_with_email_token(token, second_factor_code):
    user = get_user_from_token(token)
    if not user or not verify_second_factor(user, second_factor_code):
        return "Autenticación fallida"
    return "Acceso concedido"
```

Este enfoque es más equilibrado porque:

  - Aplica el mismo nivel de seguridad que otros métodos (como MFA).
  - Exige que todos los flujos alternativos validen identidad de forma sólida.
  - Reduce la superficie de ataque sin sacrificar flexibilidad.

💡 Recomendación práctica

Diseñar todos los métodos de autenticación con un enfoque coherente y equilibrado. Evitar accesos simplificados que puedan ser aprovechados como puntos débiles. Asegurar que cada opción esté validada, documentada y monitoreada. Si se ofrecen accesos sociales o por enlaces, limitar su alcance o exigir pasos complementarios como validación de segundo factor o vigencia estricta del token.

# ✅ Checklist de autenticación segura
- [x] Solicitud de autenticación con respuestas genéricas (sin revelar si el usuario existe).
- [x] Validación de credenciales utilizando funciones de hash robustas como bcrypt, scrypt o Argon2.
- [x] Implementación de protocolos de autenticación estándar (OAuth2, OpenID Connect, SAML).
- [x] Interfaz definida y segura para recibir credenciales (token, contraseña, biometría).
- [x] Aplicación obligatoria de MFA en accesos críticos o privilegiados.
- [x] Evitación de autenticación basada en conocimientos triviales (preguntas de seguridad).
- [x] Asignación de factores de autenticación individuales por cuenta (no compartidos).
- [x] Definición de tiempos de expiración para tokens, sesiones y enlaces de recuperación.
- [x] Generación de tokens de recuperación únicos, temporales y de un solo uso.
- [x] Inclusión de pruebas de humanidad (CAPTCHA o similar) para prevenir ataques automatizados.
- [x] Incorporación de verificación biométrica en dispositivos compatibles.
- [x] Registro de cada acceso con detalles como IP, navegador, ubicación estimada.
- [x] Notificación al usuario tras cada nuevo acceso o intento relevante.
- [x] Igual nivel de seguridad en todos los métodos de autenticación disponibles.
- [x] Protección contra ataques de enumeración mediante respuestas uniformes.
- [x] Tiempos de respuesta indistinguibles entre éxito y fallo para evitar análisis por timing.
- [x] Prevención de bloqueos definitivos por intentos fallidos repetidos (uso de delays progresivos).
- [x] Mecanismos claros para invalidar sesiones y revocar dispositivos sospechosos.

> Este checklist resume prácticas clave. Su implementación ayuda a reducir riesgos comunes en el manejo de la autenticación.

# 📚 Referencias técnicas

## 🧪 Fluid Attacks

- [Assign MFA mechanisms to a single account](https://help.fluidattacks.com/portal/en/kb/articles/criteria-requirements-362)  
- [Request access credentials](https://help.fluidattacks.com/portal/en/kb/articles/criteria-requirements-229)  
- [Proper authentication responses](https://help.fluidattacks.com/portal/en/kb/articles/criteria-requirements-225)  
- [Establish authentication time](https://help.fluidattacks.com/portal/en/kb/articles/criteria-requirements-236)  
- [Establish safe recovery](https://help.fluidattacks.com/portal/en/kb/articles/criteria-requirements-238)

## 🧱 CWE (Common Weakness Enumeration)

- [CWE-287: Improper Authentication](https://cwe.mitre.org/data/definitions/287.html)  
- [CWE-307: Improper Restriction of Excessive Authentication Attempts](https://cwe.mitre.org/data/definitions/307.html)  
- [CWE-384: Session Fixation](https://cwe.mitre.org/data/definitions/384.html)  
- [CWE-798: Use of Hard-coded Credentials](https://cwe.mitre.org/data/definitions/798.html)  
- [CWE-295: Improper Certificate Validation](https://cwe.mitre.org/data/definitions/295.html)

## 🔐 OWASP (Open Worldwide Application Security Project)

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)  
- [OWASP Multifactor Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Multifactor_Authentication_Cheat_Sheet.html)  
- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)  
- [OWASP Top 10 - A07:2021 – Identification and Authentication Failures](https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/)  
- [OWASP ASVS – V2: Authentication Verification Requirements](https://github.com/OWASP/ASVS)

## 🧩 Sonar (SonarQube / SonarCloud Rules)

- [S2068: Hardcoded credentials (Sonar rule)](https://rules.sonarsource.com/java/RSPEC-2068/)  
- [S3854: Use of weak hashing for passwords or tokens](https://rules.sonarsource.com/java/RSPEC-3854/)  
- [S6351: Session expiration not handled](https://rules.sonarsource.com/javascript/RSPEC-6351/)  
- [S4434: Untrusted input in authentication process](https://rules.sonarsource.com/javascript/RSPEC-4434/)  
- [Security Hotspots – Authentication Controls](https://docs.sonarsource.com/sonarqube/latest/user-guide/security-hotspots/)
