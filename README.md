# ALLOPS – Flujo de alta de proyectos CapEx (Power Automate)

Automatización del alta de proyectos de inversión (CapEx) en ALLOPS utilizando Power Automate, SharePoint Online y Microsoft 365.  
El flujo crea la estructura documental base del proyecto, genera enlaces útiles (Gantt, MS Project, cost tracking, dashboard, carpeta compartida, etc.) y envía un correo automático de bienvenida al responsable del proyecto.

Desarrollado para empresa multinacional para el uso en la región LATAM.

> **Nota:** En este repositorio se usa el nombre **ALLOPS** para referirse al sitio corporativo de SharePoint donde se centralizan los proyectos CapEx. El flujo puede adaptarse a cualquier entorno similar cambiando las listas y rutas de SharePoint.

---

## Objetivo

Reducir el trabajo manual al crear un nuevo proyecto CapEx:

- Estandarizar la estructura de carpetas y archivos iniciales.
- Centralizar la información del proyecto en una lista maestra de SharePoint.
- Proveer al usuario final todos los enlaces clave en un solo correo automático.
- Facilitar el seguimiento de costos y la planificación (Gantt / MS Project / dashboard).
- Proveer un medio de comunicación (canal de MS Teams) para todos los miembros del proyecto y el equipo de ingeniería.

---

## Tecnologías utilizadas

- **Power Automate (cloud)**
- **SharePoint Online**
- **Outlook 365** (envío de correos)
- **REST API de SharePoint** (`CreateCopyJobs`) para copiar plantillas de carpetas/archivos.
- **REST API de Microsoft Graph** para crear canal y publicaciones de bienvenida.
  
---

## Arquitectura general

Flujo de alto nivel:

1. **Disparador**  
   - When an item is created... Se genera un envío de un formulario de 3 campos en sitio de SharePoint (completado por el PM del proyecto).
   - Se reciben campos como: país / compañía, ID interno del proyecto, nombre del proyecto, nombre del solicitante, complejidad, etc.
   - El flujo parent le otorga un ID único al elemento contando los elementos de la lista y sumándole 1 (con iteración incorporada para que no falle)

2. **Inicialización de variables de configuración**  
   - Objeto `cfg` con enlaces base del proyecto (por ejemplo `varLink` a la ficha del proyecto en sitio de SharePoint).
   - Variable `varError` para verificar si falla y avisar donde (cada rama del flujo está dentro de un scope).

3. **Creación de estructura documental**  
   - Acción HTTP → `/_api/site/CreateCopyJobs?copy=true`  
   - Copia de carpetas/archivos desde una ruta de **templates** a una nueva ruta específica del proyecto.

4. **Actualización / creación de item en lista maestra**  
   - Lista SharePoint (por ejemplo: `AR Projects Master list`).
   - Se almacenan metadatos del proyecto y los principales enlaces generados.

5. **Envío de correo automático al responsable**  
   - Asunto dinámico:  
     `@{triggerBody()?['text_3']}_@{triggerBody()?['text_1']}_@{triggerBody()?['text_4']}`
   - Cuerpo HTML con:
     - Saludo personalizado usando el **primer nombre** del creador (PM).
     - Enlace principal al proyecto en sitio de SharePoint principal.
     - Enlaces a instructivos (RPA, sincronización con MS Project, actualización de cost tracking, etc.).
     - Enlaces a Gantt, MS Project, cost tracking, dashboard, carpeta compartida y carpeta azul.

6. **Registro / manejo básico de errores**  
   - Acciones de registro (log) en caso de fallo de la API de copia o de actualización en la lista.

Para más detalle, ver [`docs/arquitectura.md`](docs/arquitectura.md).

---

## Estructura del repositorio

```text
.
├── README.md                 # Descripción general del proyecto
├── flows/
│   └── <CPXProvisioningChild-292B8380-6DC1-F011-BBD2-6045BD9F321D>.json   # Export del flujo de Power Automate CHILD
│   └── <CPXinitializationARParent-CFFF159C-D4CA-F011-8544-7CED8D592B6B>.json  # Export del flujo de Power Automate PARENT
└── docs/
    └── arquitectura.md       # Documentación técnica de la arquitectura del flujo
