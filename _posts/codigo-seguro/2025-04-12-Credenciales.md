---
title: "Código seguro - Credenciales"
classes: wide
header:
  teaser: /assets/images/clasoxyjk07nb0lo1gg0j8xg4.png
ribbon: orange
description: "Recomendaciones clave para aplicar buenas prácticas de código seguro en la gestión e implementación de credenciales"
categories:
  - Código seguro
tags:
  - codigo-seguro
toc: true
---


# Introduction

El uso adecuado de credenciales es un componente esencial en el desarrollo de software seguro. No se trata solo de elegir contraseñas largas o complejas, sino de aplicar criterios técnicos precisos durante todo su ciclo de vida: generación, almacenamiento, uso, rotación y eliminación.

En esta sección del blog, "Código Seguro", vamos a recorrer una serie de buenas prácticas para diseñar e implementar sistemas que gestionen credenciales de forma responsable y alineada con los estándares actuales. Algunos de los temas que trataremos incluyen:

  - Uso de salts aleatorios y almacenamiento de contraseñas con algoritmos de hash adecuados.

  - Longitud mínima y construcción correcta de contraseñas y passphrases.

  - Control de cambios frecuentes, reutilización y caducidad.

  - Mecanismos para gestionar credenciales temporales y tokens de un solo uso (OTP).

  - Herramientas de gestión de contraseñas y criterios para su integración.

  - Políticas específicas para accesos de terceros, cuentas inactivas y credenciales por defecto.

Esta guía no está orientada únicamente a cumplir requisitos normativos, sino a incorporar prácticas de diseño seguro desde el inicio del desarrollo. 

## Define una herramienta de gestión de contraseñas

Las `credenciales` de acceso, especialmente aquellas asociadas a usuarios con permisos elevados, deben ser administradas a través de herramientas especializadas. Usar métodos manuales o almacenar contraseñas en archivos de configuración, planillas o código fuente representa una amenaza directa a la seguridad de cualquier sistema.

Las herramientas de gestión de contraseñas como `HashiCorp Vault`, `Bitwarden`, `1Password CLI`, `KeePassXC` o `LastPass` permiten centralizar y controlar el ciclo de vida completo de las contraseñas: desde su creación hasta su rotación, expiración o eliminación. Estas herramientas ofrecen cifrado fuerte, registro de accesos, control granular por roles, y compatibilidad con integraciones automatizadas (por ejemplo, en pipelines de `CI/CD`).

❌ Ejemplo de código inseguro

```
# Inseguro: contraseña escrita en texto plano dentro del código
SMTP_PASSWORD = "SuperSecret2024!"
```

✅ Ejemplo de código seguro

```
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
