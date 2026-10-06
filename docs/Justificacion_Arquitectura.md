# Justificación de la estructura del proyecto

**Proyecto:** Sistema de Gestión Documental GAD Amaguaña
**Tecnología:** C# / .NET 10 LTS / WPF / PostgreSQL / Active Directory
**Patrón de presentación:** MVVM (Model-View-ViewModel)
**Arquitectura:** Por capas (Domain, Application, Infrastructure, Desktop)

---

## 1. Criterios de diseño

La estructura responde a cinco criterios. Cada directorio se justifica con al menos uno de ellos.

| # | Criterio | Qué significa en este proyecto |
|---|---|---|
| C1 | **Separación de responsabilidades** | Cada carpeta tiene una sola razón para cambiar. La pantalla no sabe cómo se guarda un archivo, y la base de datos no sabe cómo se ve una ventana. |
| C2 | **Regla de dependencias** | Las dependencias apuntan hacia el centro: `Desktop → Application → Domain`. El dominio no depende de nada. |
| C3 | **Capacidad de prueba** | Las reglas del GAD (por ejemplo, el nombre normalizado) deben poder probarse sin abrir la interfaz, sin AD y sin base de datos. |
| C4 | **Trazabilidad con el SRS** | Cada carpeta funcional se asocia a uno o más requisitos (RF, RNF, RN) para poder demostrar que el SRS se implementó. |
| C5 | **Facilidad de reemplazo** | Si cambia Active Directory, el motor de base de datos o el modo de integración con el Explorador, el cambio se limita a `Infrastructure`. |

---

## 2. Vista general

```
GestorDocumental/
├── GestorDocumental.sln
├── Directory.Build.props
├── .editorconfig
├── .gitignore
├── README.md
│
├── docs/
├── database/
├── scripts/
│
├── src/
│   ├── GestorDocumental.Domain/
│   ├── GestorDocumental.Application/
│   ├── GestorDocumental.Infrastructure/
│   └── GestorDocumental.Desktop/
│
└── tests/
    ├── GestorDocumental.Domain.Tests/
    ├── GestorDocumental.Application.Tests/
    └── GestorDocumental.Infrastructure.Tests/
```

Regla de dependencias entre proyectos:

```
Desktop ──► Application ──► Domain
   │             ▲
   └──► Infrastructure ──┘
```

---

## 3. Archivos de la raíz

### `GestorDocumental.sln`
- **Objetivo:** agrupar todos los proyectos en una sola solución.
- **Justificación:** permite abrir, compilar y probar todo el sistema desde Rider con un solo archivo. Sin la solución, cada proyecto debería gestionarse por separado y las referencias entre ellos se perderían.

### `Directory.Build.props`
- **Objetivo:** centralizar la configuración común de compilación (versión de .NET, manejo de nulos, analizadores de código).
- **Justificación (C1):** evita repetir la misma configuración en siete archivos `.csproj`. Si se actualiza la versión de .NET, se cambia en un solo lugar.

### `.editorconfig`
- **Objetivo:** definir reglas de estilo de código (nombres, indentación, orden).
- **Justificación:** mantiene un código uniforme aunque trabajen varias personas, y reduce discusiones de estilo en las revisiones.

### `.gitignore`
- **Objetivo:** excluir del control de versiones los archivos generados (`bin/`, `obj/`) y los sensibles.
- **Justificación:** protege que no se suban contraseñas, cadenas de conexión reales ni archivos de usuario (RNF-01, credenciales fuera del código).

### `README.md`
- **Objetivo:** explicar cómo compilar, configurar y ejecutar el proyecto.
- **Justificación:** es la primera página que ve cualquier persona que reciba el proyecto (evaluadores, el equipo de Sistemas del GAD, futuros mantenedores).

---

## 4. Carpetas de apoyo

### `docs/`
- **Objetivo:** reunir toda la documentación del proyecto.
- **Rol:** respaldo documental y trazabilidad.
- **Justificación (C4):** mantener el SRS junto al código asegura que ambos evolucionen a la vez. Se organiza en:

| Subcarpeta | Contenido | Justificación |
|---|---|---|
| `docs/arquitectura/` | Diagramas de capas, de despliegue y de flujo (clic derecho → registro) | Sustenta las decisiones de diseño y facilita la defensa del proyecto. |
| `docs/despliegue/` | Manual de instalación y configuración del servidor (Apéndice A del SRS) | El Departamento de Sistemas necesita instrucciones claras para configurar NTFS, ABE y auditoría (RNF-05). |

### `database/`
- **Objetivo:** guardar los scripts SQL de PostgreSQL.
- **Rol:** definir y reproducir el esquema de la base de datos.
- **Justificación:** el esquema debe poder recrearse en cualquier servidor sin depender de la memoria de nadie. Es además donde se define el **usuario de aplicación con permisos mínimos** (RNF-01, criterio 3).

| Subcarpeta | Contenido | Justificación |
|---|---|---|
| `database/scripts/` | Creación de tablas, índices de búsqueda (RF-02), roles y permisos | Permite desplegar la base de datos de forma repetible y revisable. |
| `database/seed/` | Datos iniciales: tipos de documento, departamentos | El modal de carga (RF-03) necesita estas listas desde el primer día. |

### `scripts/`
- **Objetivo:** automatizar tareas operativas.
- **Rol:** instalación, desinstalación y publicación.
- **Justificación (RF-04, RNF-04):** la integración con el Explorador (menú contextual y "Enviar a") debe instalarse **por usuario y sin administrador**. Hacerlo a mano en cada equipo es lento y propenso a errores.

| Archivo | Objetivo | Requisito |
|---|---|---|
| `instalar-integracion-explorador.ps1` | Registra el menú contextual en `HKCU` y el acceso en `shell:sendto` | RF-04, RNF-04, Apéndice B |
| `desinstalar-integracion-explorador.ps1` | Elimina esas entradas sin dejar residuos | RNF-04 |
| `publicar.ps1` | Genera la versión distribuible de la aplicación | RNF-04 |

---

## 5. Código fuente: `src/`

### 5.1 `GestorDocumental.Domain`

- **Objetivo:** contener las reglas y los conceptos propios del GAD.
- **Rol:** el núcleo del sistema.
- **Justificación (C2, C3):** no referencia a ningún otro proyecto, ni a WPF, ni a PostgreSQL, ni a Windows. Por eso es el proyecto más estable y el más fácil de probar. Si todo lo demás cambiara, estas reglas seguirían siendo válidas.

| Subcarpeta | Objetivo | Justificación | Requisito |
|---|---|---|---|
| `Entities/` | Clases del negocio: `Documento`, `EventoAuditoria`, `Usuario`, `Departamento`, `TipoDocumento` | Representan los conceptos que el SRS ya nombra. Se definen una vez y todas las capas los comparten. | RF-03, RF-06 |
| `ValueObjects/` | Valores con reglas propias: `NombreNormalizado`, `RutaUnc`, `Secuencial`, `HashArchivo` | Un nombre normalizado no es un simple texto: tiene reglas. Encapsularlas impide crear valores inválidos en cualquier parte del sistema. | RN-01 a RN-05, RN-08 |
| `Enums/` | `TipoEvento` (login, búsqueda, carga, apertura, acceso denegado) y `ResultadoEvento` | Evita escribir textos libres para los tipos de evento; cada registro de auditoría usa un valor controlado. | RF-06 |
| `Rules/` | Las reglas de negocio hechas código, como el normalizador que genera `OFICIO_FINANCIERO_2026_00142.pdf` | Concentra RN-01 a RN-08 en un solo lugar. Si el GAD cambia el patrón de nombre, solo se modifica aquí. | RN-01 a RN-08 |
| `Exceptions/` | Errores del negocio: `NombreDuplicadoException`, `AccesoDenegadoException` | Permiten distinguir un error del negocio (nombre repetido) de un error técnico (red caída) y mostrar el mensaje correcto al usuario. | RN-06, RF-05 |

### 5.2 `GestorDocumental.Application`

- **Objetivo:** orquestar lo que el sistema hace (registrar, buscar, abrir, auditar).
- **Rol:** casos de uso.
- **Justificación (C1, C5):** define **qué** se hace sin saber **cómo** se hace. Habla con AD, archivos y base de datos únicamente a través de interfaces, lo que permite cambiar o simular esas piezas.

| Subcarpeta | Objetivo | Justificación | Requisito |
|---|---|---|---|
| `Abstractions/` | Interfaces: `IAutenticacionService`, `IRepositorioArchivos`, `IDocumentoRepository`, `IAuditoriaService`, `IPermisosService` | Es el punto que desacopla el sistema. `Application` pide "guardar un documento" sin saber que se usa PostgreSQL. Habilita las pruebas con simulaciones (C3). | Todos |
| `UseCases/Autenticacion/` | Validar la sesión del usuario | Aísla la lógica de acceso, incluido el caso de abrir la app desde el Explorador sin sesión activa. | RF-01, RNF-01 |
| `UseCases/RegistrarDocumento/` | Validar, normalizar el nombre, copiar el archivo y guardar los metadatos | Es el caso de uso central del sistema y toca varias reglas a la vez; merece su propio espacio. | RF-03, RF-04, RN-01 a RN-09 |
| `UseCases/BuscarDocumentos/` | Buscar y filtrar documentos según permisos | Combina criterios (lógica AND) y filtra por lo que el usuario puede leer. | RF-02, RF-05 |
| `UseCases/AbrirDocumento/` | Abrir un documento y mostrar su ubicación | Cada apertura debe auditarse; centralizarlo garantiza que no se olvide. | RF-08, RF-06 |
| `UseCases/Auditoria/` | Registrar y consultar eventos | Separa la auditoría del resto para poder tratarla de forma asíncrona y fiable. | RF-06, RNF-02, RNF-06 |
| `DTOs/` | Objetos simples para pasar datos entre capas (por ejemplo, `ResultadoBusquedaDto`) | Evita exponer las entidades del dominio directamente a la interfaz y permite enviar solo los datos necesarios. | RF-02 |
| `Validators/` | Validación del modal: campos obligatorios, formato del secuencial | Las validaciones viven fuera de la pantalla, por lo que se aplican igual sin importar desde dónde se invoque el caso de uso. | RF-03 |

### 5.3 `GestorDocumental.Infrastructure`

- **Objetivo:** implementar las interfaces de `Application` con tecnologías reales.
- **Rol:** conexión con el mundo exterior (AD, archivos, PostgreSQL, registro de Windows).
- **Justificación (C5):** todo lo que depende de un servicio externo está aquí. Si el GAD cambia de motor de base de datos o de dominio, solo se modifica este proyecto.

| Subcarpeta | Objetivo | Justificación | Requisito |
|---|---|---|---|
| `ActiveDirectory/` | Autenticación por Kerberos/LDAPS; lectura de grupos y departamento del usuario | Encierra todo el código de dominio en un solo lugar. Usa protocolos seguros y no guarda contraseñas. | RF-01, RF-05, RNF-01 |
| `FileSystem/` | Copiar archivos a rutas UNC, calcular el hash, leer permisos NTFS, vigilar la carpeta de entrada | Agrupa todo el acceso al servidor de archivos. Incluye las particularidades de SMB (por ejemplo, que `FileSystemWatcher` pierde eventos y requiere refuerzo). | RF-03, RF-05, RF-10, RN-08 |
| `Persistence/` | Acceso a PostgreSQL con EF Core y Npgsql | Contiene todo lo relacionado con la base de datos. Se divide en tres partes (abajo). | RF-02, RF-03, RF-06 |
| `Auditoria/` | Cola en memoria (`Channel`) con un proceso en segundo plano y una cola local en disco | Garantiza que registrar la auditoría no bloquee la interfaz (menos de 200 ms) y que no se pierdan eventos si PostgreSQL no responde. | RF-06, RNF-02, RNF-06 |
| `ShellIntegration/` | Registra y quita las claves del menú contextual y de "Enviar a" en `HKCU` | Concentra la integración con el Explorador (Modelo B), la pieza que diferencia este proyecto. Trabajar sobre `HKCU` evita pedir permisos de administrador. | RF-04, RNF-04 |
| `DependencyInjection.cs` | Registra todas las implementaciones en un único punto | Permite que `Desktop` configure el sistema completo con una sola llamada y deja claro qué implementación se usa para cada interfaz. | Todos |

Subcarpetas de `Persistence/`:

| Subcarpeta | Objetivo | Justificación |
|---|---|---|
| `Configurations/` | Mapeo de entidades a tablas | Mantiene las entidades del dominio limpias de detalles de base de datos. |
| `Repositories/` | Guardar y buscar documentos y eventos | Es la implementación de las interfaces `IDocumentoRepository` y similares. Aquí viven las consultas de búsqueda (índices `pg_trgm` y *full-text search*). |
| `Migrations/` | Historial versionado del esquema | Permite actualizar la base de datos entre versiones de forma controlada. |

### 5.4 `GestorDocumental.Desktop`

- **Objetivo:** mostrar las pantallas y recibir las acciones del usuario.
- **Rol:** capa de presentación. Aquí vive el patrón **MVVM**.
- **Justificación (C1):** es el único proyecto que conoce WPF. No contiene reglas del negocio: solo traduce lo que el usuario hace en llamadas a los casos de uso.

Correspondencia con MVVM:

| MVVM | Ubicación |
|---|---|
| **View** | `Desktop/Views/` |
| **ViewModel** | `Desktop/ViewModels/` |
| **Model** | `Domain/` + `Application/` + `Infrastructure/` |

| Elemento | Objetivo | Justificación | Requisito |
|---|---|---|---|
| `Program.cs` | Punto de entrada; lee los argumentos (`--subir "ruta"`) | Cuando Windows lanza la app desde el menú contextual, aquí se recibe la ruta del archivo. | RF-04 |
| `App.xaml` | Configuración global de la aplicación WPF | Carga los estilos y recursos comunes. | RNF-03 |
| `appsettings.json` | Configuración (ruta del servidor, servidor PostgreSQL) | Evita fijar valores en el código y permite ajustar cada despliegue. **No debe contener contraseñas.** | RNF-01 |
| `Startup/` | Control de **instancia única**, lectura de argumentos y arranque con inyección de dependencias | Windows lanza el programa una vez por cada archivo seleccionado; este módulo agrupa todo en una sola instancia y un solo modal. | RF-09, RF-04 |
| `Views/` | Ventanas XAML: Login, Búsqueda, ModalCarga, ModalFiltros | Son la **V** de MVVM. Contienen solo diseño y enlaces, sin lógica del negocio. | RF-01, RF-02, RF-03 |
| `ViewModels/` | Un ViewModel por cada vista | Son la **VM** de MVVM. Mantienen el estado de la pantalla y llaman a los casos de uso. Se pueden probar sin abrir la ventana. | RF-01 a RF-08 |
| `Controls/` | Componentes reutilizables: visor de PDF (WebView2), tarjeta de resultado | Evita duplicar la misma pieza visual en la búsqueda y en el modal de carga. | RF-07 |
| `Services/` | Servicios propios de la interfaz: diálogos, navegación entre ventanas, ícono de la bandeja del sistema | Mantiene los `ViewModels` limpios: ellos piden "mostrar un aviso" sin saber cómo se dibuja. | RF-09 |
| `Converters/` | Convertidores XAML (por ejemplo, fecha a texto, estado a color) | Pequeñas utilidades de presentación que no pertenecen ni a la vista ni al ViewModel. | RNF-03 |
| `Resources/Styles/` | Colores institucionales (amarillo, blanco y celeste pastel) y estilos | Cambia la apariencia en un solo lugar y asegura coherencia visual con la imagen del GAD. | 3.1.1, RNF-03 |
| `Resources/Icons/` | Íconos y logo del GAD | Centraliza los recursos gráficos. | 3.1.1 |
| `Resources/Strings/` | Textos de la interfaz y mensajes de error | Facilita corregir redacciones ("Acceso denegado", "No se encontraron documentos...") sin tocar el código. | RF-02, RF-05 |

---

## 6. Pruebas: `tests/`

- **Objetivo:** comprobar automáticamente que el sistema cumple lo especificado.
- **Justificación (C3, C4):** cada proyecto de pruebas refleja una capa. Así, una falla se ubica rápidamente y se demuestra que las reglas del SRS funcionan.

| Proyecto | Qué prueba | Justificación |
|---|---|---|
| `GestorDocumental.Domain.Tests` | El normalizador de nombres y las reglas RN-01 a RN-08 | Es lo más importante de probar: un nombre mal generado afecta a todos los expedientes. No necesita AD ni base de datos. |
| `GestorDocumental.Application.Tests` | Los casos de uso, con AD y archivos simulados | Verifica los flujos completos (por ejemplo, rechazar un documento sin permiso de escritura) sin depender de infraestructura real. |
| `GestorDocumental.Infrastructure.Tests` | Pruebas de integración contra una base PostgreSQL de pruebas | Comprueba que las consultas de búsqueda y el guardado de auditoría funcionan sobre el motor real. |

---

## 7. Matriz de trazabilidad (resumen)

| Requisito | Dónde se implementa |
|---|---|
| RF-01 Autenticación | `Application/UseCases/Autenticacion`, `Infrastructure/ActiveDirectory`, `Desktop/Views` (Login) |
| RF-02 Búsqueda | `Application/UseCases/BuscarDocumentos`, `Infrastructure/Persistence/Repositories`, `Desktop/Views` (Búsqueda, ModalFiltros) |
| RF-03 Carga y normalización | `Domain/Rules`, `Application/UseCases/RegistrarDocumento`, `Infrastructure/FileSystem`, `Desktop/Views` (ModalCarga) |
| RF-04 Integración con el Explorador | `Infrastructure/ShellIntegration`, `Desktop/Program.cs`, `scripts/` |
| RF-05 Control de acceso | `Infrastructure/ActiveDirectory`, `Infrastructure/FileSystem`, `Application/Abstractions` |
| RF-06 Auditoría | `Application/UseCases/Auditoria`, `Infrastructure/Auditoria`, `database/scripts` |
| RF-07 Previsualización | `Desktop/Controls` |
| RF-08 Apertura de documentos | `Application/UseCases/AbrirDocumento` |
| RF-09 Instancia única y bandeja | `Desktop/Startup`, `Desktop/Services` |
| RF-10 Carpeta de entrada | `Infrastructure/FileSystem` |
| RNF-01 Seguridad | `Infrastructure/ActiveDirectory`, `database/scripts` (usuario con permisos mínimos) |
| RNF-02 y RNF-06 Rendimiento y fiabilidad de la auditoría | `Infrastructure/Auditoria` |
| RNF-04 Compatibilidad e instalación | `scripts/`, `Infrastructure/ShellIntegration` |
| RNF-05 Auditoría complementaria del servidor | `docs/despliegue` (configuración del servidor) |
| RN-01 a RN-08 Reglas de nombre | `Domain/Rules`, `Domain/ValueObjects`, `tests/Domain.Tests` |

---

## 8. Nota sobre una versión simplificada

Si el tiempo del proyecto es limitado, `Domain` y `Application` pueden fusionarse en un solo proyecto (manteniendo sus carpetas internas), quedando tres proyectos en lugar de cuatro. Lo esencial es conservar la separación entre las **interfaces** (qué se necesita) y sus **implementaciones** (cómo se hace), y que los `ViewModels` nunca accedan directamente a archivos, AD o la base de datos.