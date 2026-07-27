# DevOps y control de versiones — Guías

Cómo se gestiona el código en equipo y cómo llega de forma automática a producción: control de versiones, integración continua, contenedores y el servidor donde acaban ejecutándose.

---

## Contenido

### [Git](git/README.md)
El sistema de control de versiones: commits, ramas, merge, rebase, stash y el resto de piezas del flujo diario.

### [CI/CD](ci-cd/README.md)
Cómo se describe un pipeline y qué herramientas y técnicas llevan el código a producción de forma automática y segura.

### [Docker](docker/README.md)
Cómo funciona la contenedorización y cómo escribir y entender un `Dockerfile`.

### [Despliegue en un VPS](despliegue-en-vps/README.md)
Llevar una aplicación dockerizada a un servidor Linux propio: endurecer el acceso (SSH, firewall, fail2ban), montar el reverse proxy con HTTPS automático, separar redes y volúmenes, y dejarlo operando solo con copias de seguridad, cron y acceso por VPN.

### [Observabilidad](observabilidad/README.md)
Los tres pilares para entender qué pasa dentro de un sistema en producción: logs, métricas y trazas, y el estándar (OpenTelemetry) que los unifica.

### [Mensajería asíncrona](mensajeria-asincrona/README.md)
Cómo desacoplar servicios con colas y eventos en vez de llamadas directas, con RabbitMQ y Azure Service Bus como piezas concretas.
