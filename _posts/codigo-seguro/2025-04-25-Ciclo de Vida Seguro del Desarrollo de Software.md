---
title: "Ciclo de Vida Seguro del Desarrollo de Software (SSDLC)"
classes: wide
header:
  teaser: /assets/images/site_data/c-ssdlc.png
ribbon: blue
description: "Guía integral sobre cómo incorporar seguridad en cada fase del ciclo de desarrollo de software moderno."
categories:
  - Codigo Seguro
tags:
  - ssdlc
  - desarrollo seguro
  - devsecops
  - threat modeling
  - seguridad de aplicaciones
toc: true
---

# Introducción

El desarrollo de software moderno no puede prescindir de la seguridad. La complejidad de las arquitecturas actuales, la proliferación de amenazas y la creciente presión normativa exigen un enfoque estructurado y sistemático para proteger aplicaciones desde su concepción hasta su mantenimiento.

El **Ciclo de Vida Seguro del Desarrollo de Software (SSDLC)** propone precisamente eso: **incorporar prácticas de seguridad en cada fase del desarrollo**, desde la recolección de requisitos hasta el mantenimiento postproducción. Adoptar un SSDLC no solo reduce la superficie de ataque y el riesgo de vulnerabilidades explotables, sino que también disminuye los costos de remediación, mejora la calidad del producto y fortalece la confianza del usuario.

Este artículo parte de la serie “Código Seguro” ofrece una guía práctica para implementar un SSDLC moderno, alineado con marcos como OWASP SAMM, NIST, ISO 27034 y DevSecOps.

---

# 🧭 1. Planificación y Análisis de Requisitos

## Objetivo

Identificar desde el inicio los **activos críticos**, **riesgos asociados** y **requisitos de seguridad** que deben estar presentes en el sistema.

## Buenas prácticas

- Entrevistar stakeholders con enfoque en seguridad.
- Identificar normativas aplicables (PCI-DSS, HIPAA, GDPR, etc.).
- Establecer requisitos no funcionales explícitos (confidencialidad, integridad, disponibilidad).
- Incluir criterios de aceptación relacionados con la seguridad.
- Categorizar datos según su sensibilidad (p. ej., personales, financieros, técnicos).

## Errores comunes

- Tratar la seguridad como un requerimiento “extra”.
- No prever riesgos asociados al dominio (por ejemplo, fraude financiero en fintech).
- No consultar al equipo de seguridad desde el inicio.

## Ejemplo práctico

```markdown
✅ Historia de usuario segura:
"Como usuario, quiero que la plataforma almacene mi contraseña de forma segura para que nadie pueda acceder a ella incluso si se compromete la base de datos."

🎯 Criterios de aceptación:
- Hash con `bcrypt` + salt aleatorio.
- Validación de complejidad mínima.
- Proceso de recuperación por MFA y correo verificado.
```

---

# 🧱 2. Diseño Seguro del Sistema

## Objetivo

Proponer una **arquitectura robusta**, documentada y resiliente frente a amenazas previsibles, evitando patrones inseguros desde el diseño.

## Subtemas clave

- **Modelado de amenazas** (STRIDE, LINDDUN, PASTA).
- **Zonificación** (DMZ, backend, frontend, third-party).
- **Diagramas DFD, arquitectura en capas, zero trust**.
- **Definición temprana de controles**: cifrado, autenticación, gestión de sesiones, etc.

## Buenas prácticas

- Usar herramientas de modelado como OWASP Threat Dragon o Microsoft Threat Modeling Tool.
- Diseñar APIs con **principio de mínimo privilegio**.
- Incluir métricas de seguridad como parte del diseño técnico.

## Ejemplo real

Supongamos que diseñás un sistema de pagos: el modelado de amenazas debería identificar riesgos como:

- MitM si no hay TLS.
- Inyección en parámetros si no hay validación.
- Fraude si no hay controles de autenticación reforzada.

---

# 💻 3. Desarrollo Seguro

## Objetivo

Garantizar que el código fuente cumpla estándares de seguridad, evitando errores comunes que derivan en vulnerabilidades.

## Recomendaciones generales

- Validar y sanear todas las entradas (input validation).
- Aplicar el principio de menor privilegio (least privilege).
- No usar funciones inseguras (`eval()`, `exec()`, etc.).
- Proteger contra inyecciones SQL, XSS, LFI, CSRF, etc.
- Usar librerías mantenidas y actualizadas.
- Prohibir credenciales hardcodeadas y usar secretos seguros.

## Fragmento de código inseguro

```python
# Peligroso: SQL Injection
cursor.execute("SELECT * FROM users WHERE username = '" + username + "'")
```

## Fragmento de código seguro

```python
# Uso de parámetros preparados
cursor.execute("SELECT * FROM users WHERE username = %s", (username,))
```

## Herramientas útiles

- Linters como `eslint-plugin-security` o `bandit`.
- Reglas de análisis SAST (SonarQube, Semgrep, Checkmarx).
- Hooks en Git para bloquear commits con secretos (como `pre-commit` y `git-secrets`).

---

# 🧪 4. Pruebas de Seguridad (Testing)

## Objetivo

Detectar vulnerabilidades antes de llegar a producción mediante pruebas estructuradas.

## Tipos de pruebas

| Tipo   | Descripción | Herramientas |
|--------|-------------|--------------|
| SAST   | Analiza código fuente estático | Semgrep, SonarQube, CodeQL |
| DAST   | Prueba la app en ejecución | OWASP ZAP, Burp Suite |
| SCA    | Analiza dependencias | Snyk, Trivy, OWASP Dependency-Check |
| IAST   | Prueba instrumentada | Contrast Security |
| RASP   | Protección en tiempo real | Sqreen, AppSensor |

## Buenas prácticas

- Incluir pruebas de fuzzing y abuso lógico.
- Simular ataques de fuerza bruta, enumeración de usuarios, bypass de roles.
- Automatizar pruebas en CI/CD, pero revisar manualmente resultados críticos.

## Ejemplo: reporte falso en login

```python
# Respuesta insegura
if not user_exists(username):
    return "Usuario inexistente"
```

**Esto permite ataques de enumeración.** Lo correcto es:

```python
return "Si las credenciales son válidas, accederá al sistema"
```

---

# 🚀 5. Integración y Despliegue Continuos (CI/CD Seguros)

## Objetivo

Garantizar que la seguridad forme parte de los pipelines automáticos y no se pierda en el camino al entorno productivo.

## Buenas prácticas

- Escaneo automático de código, dependencias y secretos.
- Validación de firmas de artefactos.
- Definición de políticas de despliegue basadas en riesgo.
- Despliegue canary o blue/green para validar sin afectar a todos los usuarios.

## Fragmento de ejemplo (GitHub Actions)

```yaml
name: CI Security Pipeline

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Análisis SAST con Semgrep
        run: semgrep --config "p/owasp-top-ten" .
      - name: Análisis de dependencias
        uses: aquasecurity/trivy-action@master
```

---

# 🛡️ 6. Mantenimiento, Monitoreo y Respuesta a Incidentes

## Objetivo

Proteger la aplicación una vez desplegada, detectando y respondiendo a comportamientos anómalos.

## Buenas prácticas

- Implementar **observabilidad** (trazabilidad, métricas, logs).
- Usar herramientas SIEM o alertas personalizadas.
- Aplicar parches de seguridad de forma continua.
- Revisar sesiones activas y dispositivos autenticados.
- Tener un plan de respuesta a incidentes formalizado.

## Ejemplo: detección de acceso sospechoso

```python
if login_ip != known_ip or device != known_device:
    send_alert(user_email, "Nuevo acceso detectado desde ubicación inusual.")
```

---

# 📋 Checklist SSDLC (Ciclo Completo)

- [x] Requisitos de seguridad definidos en la planificación.
- [x] Modelado de amenazas durante el diseño.
- [x] Principios de diseño seguro aplicados.
- [x] Revisión de código estática automatizada (SAST).
- [x] Escaneo de dependencias (SCA).
- [x] Pruebas dinámicas (DAST).
- [x] Escaneo en pipelines CI/CD.
- [x] Gestión segura de secretos.
- [x] Pruebas manuales de lógica de negocio.
- [x] Plan de respuesta a incidentes.
- [x] Monitoreo activo de logs, alertas y anomalías.
- [x] Parcheo y mantenimiento regular de librerías.

---

# 📚 Referencias técnicas

## 📖 OWASP

- [OWASP SAMM](https://owaspsamm.org/)
- [OWASP DevSecOps Guideline](https://owasp.org/www-project-devsecops-guideline/)
- [OWASP ASVS](https://github.com/OWASP/ASVS)
- [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)

## 📌 NIST

- [SP 800-64](https://csrc.nist.gov/publications/detail/sp/800-64/rev-2/final): Security Considerations in the System Development Life Cycle
- [NIST SSDF (Secure Software Development Framework)](https://csrc.nist.gov/publications/detail/white-paper/2022/02/04/ssdf/final)

## 🔧 Herramientas

- [Semgrep](https://semgrep.dev/)
- [OWASP ZAP](https://www.zaproxy.org/)
- [Trivy](https://aquasecurity.github.io/trivy/)
- [Snyk](https://snyk.io/)
- [GitLeaks](https://github.com/zricethezav/gitleaks)

---

> Si querés sumar SSDLC a tu flujo de trabajo, lo primero es cambiar el mindset: la seguridad no es una etapa, es una cultura.