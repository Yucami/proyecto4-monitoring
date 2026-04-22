# Proyecto 4 — Monitoring: Amazon EC2 + Nginx + Amazon CloudWatch

Instancia EC2 con Nginx sirviendo una página web, con monitorización completa mediante Amazon CloudWatch: métricas de sistema, logs de Nginx, alarmas y dashboard unificado.

Acceso a la instancia mediante **AWS Systems Manager Session Manager**, sin SSH, sin puerto 22 expuesto y sin key pairs.

## Servicios utilizados

- **Amazon EC2**: servidor virtual donde corre Nginx
- **Nginx**: servidor web que recibe peticiones HTTP
- **Amazon CloudWatch**: recoge métricas, logs, alarmas y dashboard
- **CloudWatch Agent**: proceso instalado en EC2 que envía métricas y logs a CloudWatch
- **AWS Systems Manager Session Manager**: acceso a la terminal de EC2 sin SSH
- **Amazon SNS**: envío de notificaciones por email cuando saltan las alarmas

## Infraestructura

| Recurso | Valor |
|---------|-------|
| Región | us-east-1 |
| Instancia | t2.micro — Amazon Linux 2023 |
| IAM Role | ec2-cloudwatch-role |
| Security Group | proyecto4-sg (puerto 80 abierto, puerto 22 cerrado) |
| Namespace CloudWatch | Proyecto4/EC2 |
| SNS Topic | proyecto4-alertas |

### Políticas IAM del rol ec2-cloudwatch-role

```
CloudWatchAgentServerPolicy     → enviar métricas y logs a CloudWatch
AmazonSSMManagedInstanceCore    → acceso vía Session Manager
```

## Decisión de seguridad: Session Manager en lugar de SSH

Acceso a EC2 mediante **AWS Systems Manager Session Manager**. Puerto 22 no expuesto. Sin key pairs.

| | SSH tradicional | Session Manager |
|--|--|--|
| Puerto 22 expuesto | ✅ necesario | ❌ no necesario |
| Key pair .pem | ✅ necesario | ❌ no necesario |
| Acceso auditado en CloudWatch | ❌ | ✅ |
| Práctica en producción | Legacy | Recomendado por AWS |

Session Manager elimina vectores de ataque comunes como fuerza bruta en el puerto 22 o robo de claves .pem, y registra todas las sesiones automáticamente en CloudWatch.

## CloudWatch Agent

EC2 por defecto solo reporta CPU a CloudWatch. El agente es necesario para enviar métricas de memoria y disco, que no están disponibles sin él.

**Métricas recogidas:**

```
mem_used_percent    → % de memoria RAM en uso
disk_used_percent   → % de disco en uso (partición /)
```

**Logs enviados a CloudWatch:**

```
/var/log/nginx/access.log  →  log group: /proyecto4/nginx/access
/var/log/nginx/error.log   →  log group: /proyecto4/nginx/error
```

## Alarmas

| Alarma | Métrica | Condición | Acción |
|--------|---------|-----------|--------|
| proyecto4-cpu-alta | CPUUtilization | > 80% durante 5 min | Email vía SNS |
| proyecto4-instancia-caida | StatusCheckFailed | > 0 durante 5 min | Email vía SNS |

`StatusCheckFailed` combina dos checks que AWS ejecuta cada minuto: uno a nivel de hardware del hypervisor y otro a nivel del sistema operativo de la instancia.

## Dashboard

Nombre: `proyecto4-dashboard`

Vista unificada con cuatro widgets:
- CPUUtilization
- mem_used_percent
- disk_used_percent
- Logs de acceso Nginx en tiempo real (CloudWatch Logs Insights)

## Por qué CloudWatch

- Es el servicio nativo de observabilidad en AWS, integrado con todos los servicios
- Centraliza métricas, logs y alarmas en un solo lugar
- No requiere infraestructura adicional ni gestión de herramientas externas

## Cuándo NO usar CloudWatch

- Para logs muy voluminosos con necesidad de búsqueda avanzada y análisis complejo, Amazon OpenSearch es más adecuado
- Para monitorización multi-cloud (AWS + GCP + Azure), herramientas como Datadog o Grafana son mejor opción

## Coste estimado

| Recurso | Coste |
|---------|-------|
| EC2 t2.micro | Free Tier (750h/mes durante 12 meses) |
| CloudWatch métricas básicas (CPU) | Gratis |
| CloudWatch métricas custom (memoria, disco) | ~$0.30/métrica/mes |
| CloudWatch Logs ingestión | $0.50/GB |
| CloudWatch Logs almacenamiento | $0.03/GB/mes |
| Alarmas (2) | Gratis (primeras 10) |
| Dashboard (1) | Gratis (primeros 3) |
| Session Manager | Gratis en EC2 |
| SNS notificaciones email | Gratis (primeras 1.000/mes) |

Para este proyecto el coste es prácticamente $0 dentro del Free Tier.

## Estructura del proyecto

```
proyecto4-monitoring/
├── README.md
└── .gitignore
```