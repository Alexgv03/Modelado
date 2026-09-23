# FAST API vs DJANGO

## Tabla comparativa

| Criterio Técnico | Django + Django REST Framework | FastAPI | Por lo tanto… |
| :--- | :--- | :--- | :--- |
| **Asincronía y Concurrencia** | Soporta vistas asíncronas pero sus componentes internos (como el ORM relacional) son síncronos por defecto.<br><br>Requiere configuraciones ASGI específicas para evitar bloqueos de hilos de ejecución. | Nativamente es asíncrono, basado en Starlette y Uvicorn.<br><br>Gestiona múltiples llamadas en paralelo sin esfuerzo de configuración extra. | FastAPI es superior para disparar consultas concurrentes a Spotify, Ticketmaster y Last.fm en un solo ciclo de espera I/O. |
| **Validación de Datos** | Utiliza Serializers de DRF. Son robustos y acoplados a modelos de datos, pero el procesamiento y la sintaxis son más pesados. | Utiliza Pydantic v2 integrado de forma nativa.<br><br>Validación basada en tipado puro de Python, con alto rendimiento de velocidad. | FastAPI nos facilita la creación de contratos de datos para moldear las respuestas dispares de las APIs de música y boleteras. |
| **Fricción de Desarrollo** | Baterías incluidas. Provee un ORM maduro, sistema de autenticación de usuarios y un Panel de Administración operativo de manera inmediata. | Micro-framework modular. No incluye ORM, sistema de usuarios ni panel de administración. El equipo debe elegir, instalar y acoplar librerías de terceros. | Django reduce el tiempo de salida al mercado al proporcionar la infraestructura base sin configurar piezas sueltas. |
| **Mitigación de Límites de Cuota** | Ofrece herramientas integradas de persistencia, sistemas de caché nativos (`django.core.cache`) y ecosistema para tareas de fondo (Celery) para almacenar datos localmente. | Requiere la integración e interconexión manual de librerías externas (SQLAlchemy/SQLModel, Alembic, fastapi-cache). | Al ser el límite de cuotas un factor crítico de las APIs externas, Django resuelve la persistencia local de contingencia de forma más rápida. |
| **Autenticación OAuth 2.0** | Compatible directamente con la librería Spotipy, la cual abstrae por completo el almacenamiento, verificación y refresco automático de los Access/Refresh Tokens. | Compatible con Spotipy en modo síncrono o requiere escribir lógica personalizada para el manejo asíncrono y almacenamiento manual de sesiones y tokens. | Django simplifica la implementación segura de la Épica de Autenticación (EPIC-01) detallada en el Backlog. |

---

## Riesgos y Límites técnicos que encontré respecto a los frameworks

### En cuanto a Django…
1. Aunque nos permita escribir vistas con `async def`, su arquitectura interna (incluyendo el ORM para conectarse a PostgreSQL y la mayoría de sus middlewares tradicionales) sigue siendo síncrona. Y si intentamos mezclar código asíncrono para las APIs y síncrono para guardar los datos en la base de datos, se genera una fricción técnica conocida como *“asynchronous/synchronous mismatch”*, lo que puede provocar bloqueos inesperados en los hilos del servidor si no lo configuramos con cuidado.
2. Es un framework pesado. Para un sistema que actúa como un API Gateway, el framework inicia componentes monolíticos que nuestro proyecto no necesita estrictamente para esta tarea (como procesador de plantillas en el servidor, middlewares de sesiones tradicionales, renderizadores de formularios masivos), lo que consumiría más memoria RAM de forma innecesaria en cada petición.

### En cuanto a FastAPI…
1. Al ser un micro-framework que no impone ninguna estructura de carpetas ni herramientas, la responsabilidad de diseñar una arquitectura limpia recae en nosotros. Sin una disciplina estricta, el código de integración de las APIs (Spotify, Ticketmaster, Last.fm) y la base de datos puede terminar disperso, acoplado y difícil de mantener a largo plazo.
2. Spotipy está diseñada para trabajar de forma síncrona. Cuando la ejecutemos dentro del bucle de eventos asíncrono de FastAPI, lo más probable es que se anulen por completo las ventajas de velocidad del framework, por lo que tendríamos que buscar librerías asíncronas alternativas menos maduras (como `async-spotify`, que por cierto ya lleva un tiempo sin actualizaciones) o a envolver las llamadas en ejecutores de hilos independientes, lo que nos aumentaría la complejidad del código base.

---

## Conflictos con las APIs

* **Spotify API y FastAPI:** Como mencionamos arriba, spotipy es síncrona. Y al ejecutarla dentro de FastAPI, va a congelar el hilo principal de procesamiento, anulando su velocidad nativa.
* **TicketMaster API y FastAPI:** TicketMaster tiene un límite de 5 peticiones por segundo. Como FastAPI es asíncrono, dispara ráfagas de consultas simultáneas que superarán este límite rápidamente, lo que va a provocarnos bloqueos con errores HTTP 429. Así que tendremos que programar un Rate limiter.
* **IP-API y Django:** IP-API limita las consultas a 45 por minuto y la arquitectura exige su respuesta antes de llamar a Ticketmaster. Si IP-API sufre lentitud o caídas, Django va a congelar sus procesos individuales esperando la respuesta, lo que puede saturar el servidor web y tirar la plataforma por completo.

---

## Conveniencia a largo plazo

### ¿Django?
1. Las actualizaciones del framework corrigen de forma masiva vulnerabilidades comunes (inyecciones, fallos de sesión) sin necesidad de revisar dependencias individuales de múltiples librerías.
2. Si la aplicación evoluciona e integra ya sea analíticas internas complejas, perfiles de usuario avanzados o herramientas de moderación para el staff, Django cuenta con las herramientas necesarias ya listas para escalar en la misma base de código.

### ¿FastAPI?
1. Requiere menor consumo de memoria RAM y CPU por petición en comparación con Django.
2. Si llegamos a decidir separar el Motor de recomendaciones del Módulo de autenticación o del Frontend, FastAPI funciona como un microservicio independiente, rápido y especializado que solo procesa datos JSON de alta velocidad sin sobrecarga innecesaria.
