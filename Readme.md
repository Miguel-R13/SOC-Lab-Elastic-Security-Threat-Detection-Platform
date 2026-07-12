# 🛡️ SOC Lab: Elastic Security Threat Detection Platform

> Construcción de un Security Operations Center completo desde cero, utilizando Elastic Stack como núcleo central de detección y respuesta. Este laboratorio documenta mis decisiones arquitectónicas, el proceso de configuración y los casos de uso reales validados en un entorno controlado.

---

## 📌 Por qué construí este laboratorio

El objetivo no era simplemente instalar herramientas: quería entender cómo funciona un SOC real por dentro. Para eso necesitaba un entorno donde pudiese generar tráfico malicioso, detectarlo, investigarlo y documentar el ciclo completo de respuesta a incidentes.

La stack elegida ElasticSearch + Kibana como SIEM/SOC. Y desde ahí comencé a crear mi EDR + NSM + CTI + ML. Cada componente fue seleccionado con criterio, no instalado por defecto intentando que se parezca a un entorno profesional.

---

## 🏗️ Arquitectura del SOC

El laboratorio está desplegado completamente en **Docker**, lo que permite reproducibilidad total y facilidad de escalado. La arquitectura se organiza en capas funcionales:

```
┌─────────────────────────────────────────────────────────────┐
│                     CAPA DE VISUALIZACIÓN                   │
│              Kibana SIEM │ Dashboards │ Alertas             │
├─────────────────────────────────────────────────────────────┤
│                   CAPA DE CORRELACIÓN                       │
│         Elasticsearch (indexación + búsqueda)               │
│         Elastic ML (detección de anomalías)                 │
├──────────────┬──────────────────────┬───────────────────────┤
│   ENDPOINTS  │       RED            │   THREAT INTEL        │
│ Elastic Agent│  Network Packet      │  Abuse.ch API         │
│ Elastic EDR  │  Capture (NSM)       │  MISP Platform        │
├──────────────┴──────────────────────┴───────────────────────┤
│                   GESTIÓN DE INCIDENTES                     │
│                   IRIS Case Management                      │
└─────────────────────────────────────────────────────────────┘
```

### Stack tecnológico principal

| Componente | Función |
|---|---|
| **Elasticsearch** | Motor de indexación, correlación y búsqueda de logs |
| **Kibana** | Visualización, SIEM y gestión de reglas de detección |
| **Fleet Server** | Control centralizado de agentes, políticas y ciclo de vida |
| **Elastic Agent** | Sensor de telemetría en endpoints (logs, métricas, seguridad) |
| **Elastic Defend** | Capa EDR activa: detección, prevención y respuesta |
| **MISP** | Plataforma de intercambio de inteligencia de amenazas |
| **IRIS** | Gestión estructurada de incidentes (case management) |

---

## 🖥️ Panel de Control: Kibana SIEM

![Panel de Elastic Kibana](https://raw.githubusercontent.com/Miguel-R13/SOC-Lab-Elastic-Security-Threat-Detection-Platform/main/screenshots/elastic-kibana.png)

---

## 🧠 Fleet Server: núcleo de gestión de agentes

El Fleet Server es el punto de control de toda la telemetría del laboratorio. Desde aquí gestiono:

- El **ciclo de vida de los Elastic Agents** desplegados en los endpoints
- La **aplicación de políticas de seguridad** de forma centralizada
- La **orquestación de la telemetría** hacia Elasticsearch

Optar por Fleet en lugar de gestión manual de agentes fue una decisión deliberada: refleja cómo se opera en entornos enterprise con decenas o cientos de endpoints.

![Fleet Server](https://raw.githubusercontent.com/Miguel-R13/SOC-Lab-Elastic-Security-Threat-Detection-Platform/main/screenshots/fleet-server.png)

---

## 🛡️ Endpoint Security (EDR)

### Política EDR diseñada para entornos Windows

Diseñé una política específica orientada a **máxima visibilidad del endpoint** sin sacrificar rendimiento. Las fuentes de datos habilitadas incluyen:

- Windows Event Logs (Security, Application, System)
- Windows Defender Events
- Authentication Events (login/logout, startup/shutdown)
- Ejecución de procesos (.exe) y actividad de PowerShell
- Modificaciones en el sistema operativo y mecanismos de persistencia

La selección no fue aleatoria: estas fuentes cubren las técnicas más comunes del marco **MITRE ATT&CK** para acceso inicial, ejecución y persistencia.

![Elastic Agent - EDR Policy](https://raw.githubusercontent.com/Miguel-R13/SOC-Lab-Elastic-Security-Threat-Detection-Platform/main/screenshots/fleet-agent-EDR-policy.png)

### Elastic Defend: EDR activo

**Elastic Defend** funciona como la capa de protección activa del endpoint, con capacidades de:

- Detección y prevención de malware en tiempo real
- Telemetría avanzada a nivel de sistema operativo
- Soporte forense para análisis post-incidente
- Respuesta activa ante amenazas confirmadas

![EDR Policy](https://raw.githubusercontent.com/Miguel-R13/SOC-Lab-Elastic-Security-Threat-Detection-Platform/main/screenshots/EDR_Policy.png)

### Vulnerability Management

Integré **CISA Known Exploited Vulnerabilities (KEV)** como fuente de inteligencia de vulnerabilidades. Esto permite identificar activos en el laboratorio expuestos a CVEs que están siendo explotados activamente en entornos reales, no solo teóricamente.

---

## 🌐 Network Security Monitoring (NSM)

Habilité captura y análisis de tráfico de red mediante **Network Packet Capture**, monitorizando protocolos seleccionados por criticidad:

| Protocolo | Razón de monitorización |
|---|---|
| DNS | Detección de C2, DNS tunneling, exfiltración |
| HTTP/TLS | Análisis de tráfico web, certificados sospechosos |
| DHCP | Identificación de nuevos activos en red |
| ICMP | Detección de escaneos y pivoting |
| MySQL | Detección de SQL Injection y accesos anómalos |

La activación selectiva de protocolos fue una decisión consciente: habilitar todo genera ruido que reduce la calidad de las detecciones.

---

## 🤖 Machine Learning: detección basada en comportamiento

En lugar de depender exclusivamente de reglas basadas en firmas, integré los **modelos nativos de Elastic ML** para detección de anomalías en tiempo real. Los casos de uso implementados son:

- **Lateral Movement Detection**: identifica patrones de movimiento lateral inusuales
- **Privileged Access Detection**: alerta sobre accesos con privilegios fuera del patrón basal
- **Data Exfiltration Detection**: detecta transferencias masivas de datos, especialmente fuera de horario

Los modelos aprenden el comportamiento normal del entorno y alertan sobre desviaciones, lo que permite detectar amenazas que no tienen firma conocida.

---

## 🧬 Threat Intelligence (CTI)

### Integración con Abuse.ch

Conecté el SOC a las siguientes fuentes de inteligencia mediante API:

- **MalwareBazaar**: samples de malware activos
- **ThreatFox**: IOCs (IPs, dominios, URLs maliciosas)
- **SSL Blacklists**: certificados asociados a infraestructura maliciosa

Retención de IOCs: **30 días**, alineada con la frecuencia de rotación de infraestructura de los actores de amenaza.

### MISP: plataforma de intercambio de inteligencia

Desplegué **MISP** como plataforma central de CTI, con una arquitectura de gobierno por niveles:

| Nivel | Responsabilidad |
|---|---|
| **L1 SOC** | Creación y documentación de eventos |
| **L2 SOC** | Análisis y enriquecimiento de indicadores |
| **L3 / Threat Intel** | Validación y publicación de inteligencia |

La integración bidireccional entre **Elastic ↔ MISP** permite enriquecer alertas automáticamente con IOCs y TTPs conocidos.

**Ejemplo práctico documentado**: intento de SQL Injection originado desde infraestructura cloud (Alibaba Cloud), con IOCs asociados y evidencia técnica completa.

![Plataforma MISP](https://raw.githubusercontent.com/Miguel-R13/SOC-Lab-Elastic-Security-Threat-Detection-Platform/main/screenshots/misp.png)

---

## 🚨 Estrategia de detección y gestión de reglas

### Filosofía: calidad sobre cantidad

Elastic Security incluye aproximadamente **1.849 reglas predefinidas**. Activarlas todas sería el error más común y más costoso en un SOC real: el volumen de alertas generado haría inviable la respuesta y llevaría al equipo a ignorar las notificaciones, un fenómeno conocido como **alert fatigue**.

La decisión fue la contraria: partir de cero y construir el conjunto de reglas activas de forma deliberada, táctica a táctica, validando cada una antes de pasar a la siguiente.

---

### Proceso de selección e implementación

**Fase 1 — Environment Baseline**

Antes de activar ninguna regla, documenté el comportamiento legítimo del entorno. Este baseline es la referencia contra la que se mide cada regla de detección — sin él, es imposible distinguir actividad anómala de ruido operacional normal.

Los siguientes elementos fueron mapeados en el endpoint Windows:

| Elemento | Baseline observado |
|---|---|
| **Procesos legítimos** | `explorer.exe`, `svchost.exe`, `lsass.exe`, `services.exe`, `winlogon.exe`, `taskhostw.exe`, `spoolsv.exe` |
| **Herramientas de administración en uso** | `powershell.exe` (interactivo, sin args codificados), `mmc.exe`, `eventvwr.exe`, `taskmgr.exe` |
| **Cuentas privilegiadas** | Cuenta de Administrador local — utilizada únicamente durante el despliegue inicial, deshabilitada posteriormente |
| **Actividad de red** | DNS hacia resolvers conocidos (8.8.8.8, 1.1.1.1), tráfico de Windows Update, sin conexiones laterales entre hosts |
| **Scheduled tasks** | Únicamente tareas por defecto de Windows (`\Microsoft\Windows\*`) — ninguna tarea personalizada |
| **Registry Run Keys** | Sin entradas fuera del software estándar de Windows y aplicaciones instaladas |
| **Creación de servicios** | Ningún servicio nuevo creado tras el despliegue inicial |
| **Horario de actividad** | Actividad del lab entre 09:00–23:00 — cualquier ejecución de procesos fuera de esta ventana se trata como anómala |

Cualquier alerta generada por los procesos o cuentas anteriores bajo condiciones normales fue clasificada como falso positivo y eliminada mediante tuning antes de pasar a la siguiente fase.

**Fase 2 — Priorización por táctica MITRE ATT&CK**

Las reglas se habilitaron por bloques, ordenados por criticidad táctica. La lógica es directa: si el atacante no puede ejecutar código ni establecer persistencia, las tácticas posteriores nunca llegan a ocurrir.

| Prioridad | MITRE Tactic | Razonamiento |
|---|---|---|
| 🔴 Crítica | Initial Access, Execution | Punto de entrada del ataque — detección lo más temprana posible |
| 🔴 Crítica | Persistence | Impide que el atacante sobreviva a un reinicio o cierre de sesión |
| 🟠 Alta | Privilege Escalation, Defense Evasion | El atacante ya está dentro y está expandiendo capacidades |
| 🟠 Alta | Lateral Movement | Contención antes de que el compromiso se extienda a otros hosts |
| 🟡 Media | Command & Control | Detección de comunicaciones salientes hacia infraestructura del atacante |
| 🟡 Media | Exfiltration | Última línea — si se llega aquí, las capas anteriores ya han fallado |

![Reglas-Elastic](https://raw.githubusercontent.com/Miguel-R13/SOC-Lab-Elastic-Security-Threat-Detection-Platform/main/screenshots/reglas.png)

**Fase 3 — Activación y observación**

Cada bloque de reglas se activó y se monitorizó durante una ventana de 48-72 horas antes de continuar. Durante ese período, las alertas generadas se clasificaron como verdaderos positivos, falsos positivos recurrentes o ruido estructural del entorno.

**Fase 4 — Tuning documentado**

Cada excepción añadida está justificada por escrito. El criterio fue consistente: una excepción se añade cuando el comportamiento es legítimo, recurrente y verificado, nunca por comodidad. Las excepciones sin documentación son deuda técnica que invalida cualquier auditoría de cumplimiento.

**Fase 5 — Revisión periódica**

Se estableció un ciclo de revisión cada 30 días para incorporar nuevas TTPs publicadas en MITRE ATT&CK, ajustar umbrales según la evolución del entorno y reactivar reglas que anteriormente generaban demasiado ruido pero que ahora tienen contexto suficiente para ser accionables.

---

### Reglas activas por táctica (selección representativa)

| MITRE Tactic | Regla | Técnica |
|---|---|---|
| Execution | PowerShell lanzado con flags `-EncodedCommand` o `-Bypass` | T1059.001 |
| Execution | Proceso hijo inusual generado desde una aplicación de Microsoft Office | T1566.001 |
| Persistence | Modificación de Registry Run Keys | T1547.001 |
| Persistence | Servicio de Windows creado por un proceso padre no estándar | T1543.003 |
| Persistence | Scheduled task creada desde línea de comandos | T1053.005 |
| Privilege Escalation | Acceso de proceso a memoria de LSASS | T1003.001 |
| Privilege Escalation | Robo o suplantación de token (Token impersonation) | T1134 |
| Defense Evasion | Windows Defender manipulado o deshabilitado | T1562.001 |
| Defense Evasion | Ejecución mediante binarios firmados — LOLBins (`certutil`, `mshta`, `regsvr32`) | T1218 |
| Lateral Movement | Ejecución remota mediante PsExec o herramientas de administración equivalentes | T1021 |
| Lateral Movement | Abuso de autenticación Pass-the-Hash / Pass-the-Ticket | T1550.002 |
| Command & Control | Beaconing regular hacia IPs externas sin hostname asociado | T1071 |
| Command & Control | Consultas DNS hacia dominios registrados recientemente | T1568 |

> Cada regla activa está mapeada en IRIS con su técnica MITRE correspondiente, el umbral configurado y las excepciones aplicadas con su justificación por escrito.

---

## 📁 Gestión de Incidentes: IRIS Case Management

Desplegué **IRIS-Web** como plataforma de gestión de incidentes, garantizando trazabilidad completa del ciclo de vida de cada alerta:

```
Detección (SIEM) → Triaje (Kibana) → Investigación (IRIS) → Contención → Cierre
```

IRIS me permite documentar:

- Evidencias recopiladas durante la investigación
- Correlación de eventos relacionados
- Timeline del incidente
- Métricas de respuesta: **MTTD** (Mean Time to Detect) y **MTTR** (Mean Time to Respond)

---

## 📊 Observabilidad de infraestructura

El SOC también monitoriza su propia salud, no solo las amenazas externas. Los componentes monitorizados incluyen:

- **Hosts**: CPU, RAM, disco de los sistemas del laboratorio
- **Red**: tráfico y latencia entre componentes
- **APIs**: health-check de servicios integrados (MISP, IRIS)
- **Certificados TLS**: alertas de expiración mediante Elastic Synthetics

---

## 🔒 Buenas prácticas implementadas

El laboratorio no solo detecta amenazas, también las aplica en su propia configuración:

| Práctica | Implementación | Framework |
|---|---|---|
| **Gestión de secretos** | Variables de entorno (.env), sin hardcoding de credenciales | CIS Controls - Control 6 |
| **Retención de logs** | Política formal de retención de datos en Elasticsearch | ISO 27001 A.12.4 |
| **Backup de configuraciones** | Protocolos de backup para índices críticos e IRIS/MISP | NIST CSF - Recover |
| **Mapeo MITRE ATT&CK** | Cada regla activa documentada contra TTPs reales | MITRE ATT&CK v14 |

---

## 🧪 Caso práctico: análisis de malware y respuesta a incidentes

Para validar la capacidad operativa del entorno, desarrollé un escenario de análisis completo que abarca el ciclo de vida entero de la respuesta a incidentes:

1. **Extracción de Indicadores de Compromiso (IoCs)**
2. **Análisis estático y dinámico en entorno aislado**
3. **Correlación con inteligencia de amenazas (MISP + Abuse.ch)**
4. **Respuesta técnica y mitigación documentada**

📄 [Ver el informe técnico completo →](https://miguel-r13.github.io/)

---

## 📐 Escalabilidad: preparado para Linux

La arquitectura está diseñada para extenderse a endpoints Linux con monitorización de:

- Logs de sistema y auditoría (`auditd`)
- Métricas de rendimiento del SO
- Telemetría de seguridad (accesos, procesos, cambios de configuración)

---

## 🗂️ Estructura del repositorio

```
SOC-Lab-Elastic-Security-Threat-Detection-Platform/
├── screenshots/                # Capturas de pantalla del entorno
│   ├── elastic-kibana.png
│   ├── fleet-server.png
│   ├── fleet-agent-EDR-policy.png
│   ├── EDR_Policy.png
│   └── misp.png
├── configs/                    # Configuraciones (sin secretos)
├── rules/                      # Reglas de detección personalizadas
└── README.md
```

---

## 🔗 Recursos relacionados

- [Caso Práctico IR: Reverse Shell & Postexplotación detectado con este SOC Lab](https://github.com/Miguel-R13/BlueTeam-Labs-WriteUps/tree/master/03-IR-Respuesta-Incidentes-Elastic)
- [Informe técnico: análisis de malware y phishing](https://miguel-r13.github.io/)
- [MITRE ATT&CK Framework](https://attack.mitre.org/)
- [Elastic Security Documentation](https://www.elastic.co/guide/en/security/current/index.html)
- [MISP Project](https://www.misp-project.org/)
- [IRIS-Web](https://dfir-iris.org/)

---

*Laboratorio construido y documentado por Miguel — Blue Team / SOC Analyst*  
*Stack: Elastic Stack · MISP · IRIS · Abuse.ch · Docker*
