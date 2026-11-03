# Red de la biblioteca · L9

## AWS · VPC 10.0.0.0/16 · us-east-1

| Subred | Rango | Zona | Ruta 0.0.0.0/0 |
|---|---|---|---|
| pública | 10.0.1.0/24 | us-east-1a | Internet Gateway |
| aplicación | 10.0.11.0/24 | us-east-1a | interfaz de red de `nat` |
| datos-a | 10.0.21.0/24 | us-east-1a | interfaz de red de `nat` |
| datos-b | 10.0.22.0/24 | us-east-1b | interfaz de red de `nat` |

| Máquina | Subred | IP privada | Tamaño | Lo que corre | Por qué ese tamaño |
|---|---|---|---|---|---|
| `gw-aws` | pública | 10.0.1.20, con Elastic IP | t3.small | Caddy con HTTPS y `biblioteca-gateway` | Gateway 30,71 MiB medidos frente a 2 GiB, con margen para Caddy y el SO. |
| `nat` | pública | 10.0.1.10 | t3.nano | NAT instance de las subredes privadas | 0,5 GiB para el SO y el reenvío de paquetes. |
| `servicios` | aplicación | 10.0.11.10 | t3.medium | `biblioteca-libros`, `biblioteca-prestamos` y `biblioteca-eventos` | Libros, Préstamos y Eventos suman 151,38 MiB medidos frente a 4 GiB. |
| `gestion` | aplicación | 10.0.11.20 | t3.medium | `biblioteca-admin`, Kafka UI, `biblioteca-simulador`, runner `aws` (Prometheus y Grafana, opcionales) | Admin y Kafka UI suman 448,45 MiB medidos frente a 4 GiB, con margen para el simulador y el runner. |
| `mq-1` | datos-a | 10.0.21.11 | t3.small | controller 1, broker 4 y `rabbit1` | Controller, broker y RabbitMQ suman 1073,5 MiB medidos frente a 2 GiB. |
| `mq-2` | datos-b | 10.0.22.12 | t3.small | controller 2, broker 5 y `rabbit2` | Controller, broker y RabbitMQ suman 1074 MiB medidos frente a 2 GiB. |
| `mq-3` | datos-a | 10.0.21.13 | t3.small | controller 3 y broker 6 | Controller y broker suman 847,6 MiB medidos frente a 2 GiB. |
| `datos` | datos-a | 10.0.21.30 | t3.small | Postgres 18 | Postgres usa 30,46 MiB medidos frente a 2 GiB, con margen para caché y el SO. |

| Grupo de reglas | Puerto | Origen | Para qué |
|---|---|---|---|
| `sg-gw` | 80 | `0.0.0.0/0` | Desafío de Let's Encrypt y redirección a HTTPS |
| `sg-gw` | 443 | `pendiente E2` | La API, solo para el BFF |
| `sg-servicios` | 3001, 3002 y 3010 | `sg-gw` y `sg-gestion` | El gateway llega a `libros` y `prestamos`; el administrador revisa `eventos` |
| `sg-mq` | 9092 | `sg-servicios`, `sg-gestion` y `sg-mq` | Clientes de Kafka y réplica entre brokers |
| `sg-mq` | 9093 | `sg-mq` | Quórum KRaft |
| `sg-mq` | 5672 | `sg-servicios` y `sg-gestion` | AMQP de RabbitMQ |
| `sg-mq` | 15672 | `sg-gestion` | Administración de RabbitMQ |
| `sg-mq` | 4369 y 25672 | `sg-mq` | Clúster de RabbitMQ |
| `sg-datos` | 5432 | `sg-servicios` y `sg-gestion` | Postgres |
| `sg-nat` | todos | `sg-servicios`, `sg-gestion`, `sg-mq` y `sg-datos` | Salida a internet de las subredes privadas |
| `sg-gestion` | ninguno | | Se entra por Session Manager |

## Azure · VNet 10.1.0.0/16 · `northcentralus`

| Subred | Rango | Salida a internet |
|---|---|---|
| pública | 10.1.1.0/24 | directa, por la IP pública de `gw-azure` |
| privada | 10.1.2.0/24 | por `gw-azure` (tabla de rutas hacia 10.1.1.4); creada con `--default-outbound false` |

| Máquina | Subred | IP privada | Tamaño | Lo que corre | Por qué ese tamaño |
|---|---|---|---|---|---|
| `gw-azure` | pública | 10.1.1.4, con IP pública estática y etiqueta `pendiente E2` | B2ats_v2 | Caddy, `biblioteca-web`, `biblioteca-gateway` y NAT de la subred privada | Web y Gateway suman 44,09 MiB medidos frente a 1 GiB, con margen para Caddy, NAT y el SO. |
| `bff` | privada | 10.1.2.4, sin IP pública | B2als_v2 | `biblioteca-bff` y el runner `azure` | BFF usa 32,63 MiB medidos frente a 2 GiB, con margen para el runner y el SO. |

| NSG | Puerto | Origen | Para qué |
|---|---|---|---|
| `gw-azure` | 443 y 80 | Internet | La web y el desafío de Let's Encrypt |
| `gw-azure` | 22 | `pendiente E2` | SSH |
| `gw-azure` | todos | 10.1.2.0/24 | El NAT de `bff` |
| `bff` | todos | 10.1.0.0/16 | Solo desde la VNet |
