## Arquitectura general

Flujo de alto nivel:

1. **Disparador**  
- Trigger: **When an item is created** en una lista de SharePoint (formulario simple de alta de proyecto).
- El formulario lo completa el **Project Manager** o responsable del proyecto.
- Campos a completar:
  - Nombre del proyecto
  - Area organizacional
  - Año fiscal
  - Planta
- El flujo **PARENT** asigna un ID único al registro de cada país (contando ítems existentes y sumando 1, con lógica de reintento para evitar colisiones).

2. **Inicialización de variables de configuración**  
- Se inicializa un objeto de configuración `cfg` con:
  - Rutas base de SharePoint donde se crearán las carpetas del proyecto.
  - Enlace a la ficha del proyecto en el sitio ALLOPS.
  - URLs de plantillas (Gantt, MS Project, cost tracking, etc.).
- Se inicializa `varError` para registrar en qué parte del flujo se produce un fallo (cada bloque principal está dentro de un **Scope** para facilitar el manejo de errores).

3. **Creación de estructura documental**  
- Acción HTTP a SharePoint:
  `POST /_api/site/CreateCopyJobs?copy=true`
- A partir de una carpeta de **templates** (estructura “modelo”), se copia:
  - La carpeta raíz del proyecto.
  - Subcarpetas por área (por ej. EHSQ, Ingeniería, Finanzas, etc.).
  - Archivos base: planilla de cost tracking, plantilla de MS Project, instructivos, etc.
- La estructura se crea en una ruta específica del proyecto, por ejemplo:  
  `/Shared Documents/ALLOPS/CPX/<Año>/<ID_Proyecto>_<Nombre>/…`

4. **Actualización / creación de item en lista maestra**  
- Se actualiza o crea un registro en una **lista maestra de proyectos** (ej.: `AR Projects Master list`).
- Se guardan:
  - Metadatos clave del proyecto (ID, país, planta, responsable, año fiscal, monto estimado, estado inicial, etc.).
  - Enlaces generados: carpeta raíz del proyecto, cost tracking, Gantt, dashboard, canal de Teams, etc.
- Esta lista funciona como **fuente única de verdad** para consultar el estado y enlaces de cualquier proyecto CapEx de la región LATAM de la empresa.
   
5. **Creación de canal en Teams y correo automático al responsable**

- Con Microsoft Graph se crea un **canal de Teams** para el proyecto (en un Team específico de ALLOPS) y se publica un mensaje de bienvenida con los enlaces principales.
- Se envía un correo al responsable del proyecto:
  - Asunto dinámico (ejemplo):  
    `@{triggerBody()?['text_3']}_@{triggerBody()?['text_1']}_@{triggerBody()?['text_4']}`  
    > Equivale a combinar nombre del proyecto, país/compañía y un ID interno.
  - Cuerpo HTML con:
    - Saludo personalizado usando el **primer nombre** del PM.
    - Enlace principal al proyecto en SharePoint.
    - Enlaces a instructivos (RPA, sincronización MS Project, actualización del cost tracking, etc.).
    - Enlaces directos a:
      - Gantt / MS Project
      - Planilla de cost tracking
      - Dashboard de seguimiento
      - Carpeta compartida del proyecto
      - Carpeta “azul” (documentación accesible para áreas no técnicas, si aplica)
        
6. **Registro y manejo básico de errores**

- Cada bloque crítico (copias de SharePoint, actualización de listas, creación de canal, envío de mail) se ejecuta dentro de un **Scope**.
- Si alguna acción falla:
  - Se actualiza `varError` indicando en qué etapa se produjo el error.
  - Se registran los detalles (por ejemplo en una lista de logs o en el propio ítem del proyecto).
  - Opcionalmente, se puede enviar un correo de alerta al equipo de Ingeniería / IT.


