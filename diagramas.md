# Diagramas del Sistema

Documentación visual y técnica del sistema:
- [Diagramas de Flujo y Navegación (Hitos 2 y 4)](./diagramas.md)
- [Modelo Entidad-Relación de Base de Datos (Hito 5)](./modelo_erd.md)

---

## 2. Hito 2: Diagramas de Flujo en Markdown / Mermaid

### 2.1 Diagrama del Ciclo de Vida del Trabajo y Procesamiento de Pago (Flujo Principal)

```mermaid
flowchart TD
    A([Inicio: Empleador crea Trabajo]) --> B[Estado: Publicado]
    B -->|Trabajador envía Postulación| C[Estado: Con Postulaciones]
    C -->|Empleador revisa perfil y Selecciona Candidato| D[Estado: Asignado]
    
    D --> E{"¿Ambos aceptan condiciones Y<br/>Trabajador completa Stripe Connect<br/>en <= 24 Horas?"}
    E -- NO --> F["Postulación pasa a EXPIRADA.<br/>Trabajo regresa a Con Postulaciones"]
    F --> C
    
    E -- SÍ --> G["Trabajador registra Llegada<br/>Ventana: Inicio -30m hasta Fin"]
    G --> H[Estado: Llegada Registrada]
    
    H --> I["Empleador realiza Pago en Stripe Connect<br/>Importe: Trabajo + 10% Comisión"]
    I --> J{"¿Cobro Confirmado por Webhook<br/>en <= 30 min ó 3 intentos?"}
    
    J -- NO --> K["Trabajo y Pago pasan a CANCELADO.<br/>Notificación a ambos usuarios"]
    K --> Z([Fin: Cancelado sin Cobro])
    
    J -- SÍ --> L["Estado Trabajo: En Curso<br/>Estado Pago: Confirmado -> Pendiente de Transferencia"]
    
    L --> M{Método de Finalización}
    
    M -- Con Código de 6 Dígitos --> N[Trabajador ingresa Código proporcionado por Empleador]
    N -->|Código Correcto| O[Estado: Finalizado]
    
    M -- Sin Código --> P["Trabajador solicita finalización + sube Evidencias"]
    P --> Q[Estado: Finalización Solicitada]
    Q --> R{"¿Empleador responde en <= 24 Horas?"}
    R -- Acepta o No Responde --> O
    R -- Impugna / Disputa --> S["Apertura de RECLAMACIÓN DE PAGO.<br/>Estado: En Reclamación"]
    
    O --> T[Sistema ejecuta Transferencia a Stripe Connect del Trabajador]
    T --> U[Estado Pago: Transferido]
    U --> V["Ambas partes emiten Calificación y Reseña 1-5 Estrellas"]
    V --> W([Fin: Trabajo Finalizado Exitosamente])
    
    S --> X["Fondos Bloqueados en Escrow.<br/>Pasa a Módulo de Casos para Administrador"]
    X --> Y([Fin: En Arbitraje Administrativo])
```

### 2.2 Diagrama del Módulo de Casos, Arbitraje de Pagos y Sanciones (Administrador)

```mermaid
flowchart TD
    A([Caso Registrado: Ticket / Reclamación / Reporte]) --> B["Estado Caso: Abierto / Pendiente"]
    B --> C[Administrador asigna caso y pasa a En Revisión]
    
    C --> D{Tipo de Caso}
    
    D -- "Reporte de Contenido / Usuario" --> E[Administrador evalúa infracción]
    E --> F{Acción de Moderación}
    F -- Contenido Abusivo --> G[Ocultar temporalmente o Retirar contenido]
    F -- Infracción de Usuario --> H["Aplicar Sanción: Advertencia / Suspensión 1,7,30d / Bloqueo / Baja Lógica"]
    F -- Sin Violación --> I[Desestimar Reporte]
    G --> J[Estado Caso: Atendido]
    H --> J
    I --> J
    
    D -- Reclamación de Pago --> K["Revisión de chat, evidencias, tiempos y motivos"]
    K --> L["Administrador registra Decisión Financiera: RESUELTO"]
    L --> M{Opciones de Liquidación}
    M -- Transferencia Total --> N1["Monto Completo -> Trabajador"]
    M -- Reembolso Total --> N2["Monto Completo + Comisión -> Empleador"]
    M -- Liquidación Parcial --> N3["Monto X -> Trabajador / Monto Y -> Empleador"]
    
    L --> O{"¿Aceptado por Solicitante O<br/>Transcurren 7 días sin Objeción?"}
    O -- "NO: Solicitante Objeta en <= 7 días" --> P["Caso regresa a EN REVISIÓN.<br/>Fondos Continúan Bloqueados"]
    P --> C
    
    O -- "SÍ: Confirmado / Cierre Automático" --> Q[Estado Caso: CERRADO]
    Q --> R[Sistema ejecuta Operación Financiera en Stripe Connect]
    R --> S{Resultado de Stripe}
    S -- Éxito --> T["Estado Pago: Transferido / Reembolsado / Liquidación Parcial Completada"]
    S -- Fallo --> U["Estado Pago: Fallido. Reintentos automáticos 3x / 24h + Ticket Técnico"]
    
    J --> V([Fin: Caso Resuelto y Cerrado])
    T --> V
```

### 2.3 Diagrama del Algoritmo de Búsqueda y Ubicación por Radio del Trabajador

```mermaid
flowchart TD
    A([Trabajador abre Explorador de Trabajos]) --> B{"¿Tiene Ubicación de Referencia Guardada?"}
    
    B -- NO --> C["Solicitar al Trabajador marcar Punto en Mapa y seleccionar Radio inicial 10 km"]
    C --> D[Guardar Punto de Referencia Inicial]
    
    B -- SÍ --> E{"¿Desea cambiar Punto de Referencia?"}
    E -- SÍ --> F{"¿Ha realizado >= 3 cambios<br/>en las últimas 24 Horas?"}
    F -- SÍ --> G[Bloquear Cambio. Mostrar mensaje de límite alcanzado]
    G --> H[Usar Ubicación de Referencia Actual]
    F -- NO --> I[Permitir marcar nuevo punto en Mapa y actualizar]
    I --> H
    
    E -- NO --> H
    D --> H
    
    H --> J[Calcular Distancia en Línea Recta Haversine entre Ubicación Referencia y Coordenadas de Trabajos Activos]
    J --> K["Filtrar Trabajos por Radio Seleccionado: 5, 10, 25 o 50 km"]
    K --> L["Aplicar Filtros Adicionales del Usuario: Categoría y Rango de Fechas"]
    
    L --> M["Ordenar Lista: Trabajos DENTRO del Radio primero por distancia ascendente"]
    M --> N[Obtener hasta 10 Trabajos adicionales FUERA del Radio]
    N --> O["Asignar a fuera de radio la etiqueta 'Fuera de Alcance'"]
    
    O --> P[Transformar Coordenadas Exactas a Rango de Distancia Estimado]
    P --> Q["Renderizar Vista: Título, Categoría, Colonia/Localidad, Rango Distancia, Evaluaciones del Empleador"]
    Q --> R([Fin: Lista de Recomendaciones Renderizada])
```

---

## 4. Hito 4: Definición del Flujo de Navegación entre Interfaces (UI Flow)

### 4.1 Diagrama de Navegación — App Móvil Empleador

```mermaid
flowchart TD
    UI_EMP_01["UI-EMP-01: Auth / Registro"] --> UI_EMP_02["UI-EMP-02: Feed / Mis Publicaciones"]
    
    UI_EMP_02 --> UI_EMP_03["UI-EMP-03: Crear / Editar Trabajo"]
    
    UI_EMP_02 --> UI_EMP_04["UI-EMP-04: Detalle Publicación / Postulantes"]
    UI_EMP_04 --> UI_EMP_05["UI-EMP-05: Perfil Candidato"]
    UI_EMP_05 -->|Seleccionar| UI_EMP_02
    
    UI_EMP_02 --> UI_EMP_06["UI-EMP-06: Trabajo Asignado & Condiciones"]
    UI_EMP_06 --> UI_EMP_07["UI-EMP-07: Checkout Stripe (Cobro)"]
    UI_EMP_06 --> UI_EMP_08["UI-EMP-08: Chat en Tiempo Real"]
    UI_EMP_06 --> UI_EMP_09["UI-EMP-09: Verificación Finalización / Código"]
    UI_EMP_09 --> UI_EMP_11["UI-EMP-11: Evaluación y Reseña"]
    UI_EMP_11 --> UI_EMP_02
    
    UI_EMP_02 --> UI_EMP_10["UI-EMP-10: Formulario Apertura de Casos"]
    UI_EMP_02 --> UI_EMP_12["UI-EMP-12: Historial Trabajos y Pagos"]
    UI_EMP_02 --> UI_EMP_13["UI-EMP-13: Perfil y Ajustes"]
```

```text
[UI-EMP-01: Auth / Registro]
       │
       ├──> [UI-EMP-02: Feed / Mis Publicaciones] <─────────────────────────────┐
       │         │                                                              │
       │         ├──> [UI-EMP-03: Crear / Editar Trabajo]                       │
       │         │                                                              │
       │         ├──> [UI-EMP-04: Detalle Publicación / Postulantes]            │
       │         │         │                                                    │
       │         │         └──> [UI-EMP-05: Perfil Candidato] ──(Seleccionar)───┤
       │         │                                                              │
       │         ├──> [UI-EMP-06: Trabajo Asignado & Condiciones]               │
       │         │         │                                                    │
       │         │         ├──> [UI-EMP-07: Checkout Stripe (Cobro)]            │
       │         │         │                                                    │
       │         │         ├──> [UI-EMP-08: Chat en Tiempo Real]                │
       │         │         │                                                    │
       │         │         └──> [UI-EMP-09: Verificación Finalización/Código]  │
       │         │                   │                                          │
       │         │                   └──> [UI-EMP-11: Evaluación y Reseña] ─────┤
       │         │                                                              │
       │         ├──> [UI-EMP-10: Formulario Apertura de Casos]                 │
       │         │                                                              │
       │         ├──> [UI-EMP-12: Historial Trabajos y Pagos]                   │
       │         │                                                              │
       │         └──> [UI-EMP-13: Perfil y Ajustes]                             │
```

---

### 4.2 Diagrama de Navegación — App Móvil Trabajador

```mermaid
flowchart TD
    UI_TRAB_01["UI-TRAB-01: Auth / Registro"] --> UI_TRAB_02["UI-TRAB-02: Explorador de Trabajos"]
    
    UI_TRAB_02 --> UI_TRAB_03["UI-TRAB-03: Ubicación Referencia y Radio Mapa"]
    
    UI_TRAB_02 --> UI_TRAB_04["UI-TRAB-04: Detalle Oportunidad"]
    UI_TRAB_04 -->|Postularse| UI_TRAB_05["UI-TRAB-05: Mis Postulaciones"]
    UI_TRAB_05 --> UI_TRAB_02
    UI_TRAB_05 --> UI_TRAB_06["UI-TRAB-06: Onboarding Stripe Connect"]
    
    UI_TRAB_02 --> UI_TRAB_07["UI-TRAB-07: Trabajo Asignado & Registrar Llegada"]
    UI_TRAB_07 --> UI_TRAB_08["UI-TRAB-08: Chat en Tiempo Real"]
    UI_TRAB_07 --> UI_TRAB_09["UI-TRAB-09: Finalizar Trabajo (Código/Evid)"]
    UI_TRAB_09 --> UI_TRAB_11["UI-TRAB-11: Evaluación Empleador"]
    UI_TRAB_11 --> UI_TRAB_02
    
    UI_TRAB_02 --> UI_TRAB_10["UI-TRAB-10: Centro de Casos y Soporte"]
    UI_TRAB_02 --> UI_TRAB_13["UI-TRAB-13: Historial Trabajos y Pagos"]
    UI_TRAB_02 --> UI_TRAB_12["UI-TRAB-12: Perfil de Trabajador"]
```

```text
[UI-TRAB-01: Auth / Registro]
       │
       ├──> [UI-TRAB-02: Explorador de Trabajos] <──────────────────────────────┐
       │         │                                                              │
       │         ├──> [UI-TRAB-03: Ubicación Referencia y Radio Mapa]           │
       │         │                                                              │
       │         ├──> [UI-TRAB-04: Detalle Oportunidad] ──(Postularse)──┐        │
       │         │                                                      │        │
       │         ├──> [UI-TRAB-05: Mis Postulaciones] <─────────────────┘        │
       │         │         │                                                     │
       │         │         └──> [UI-TRAB-06: Onboarding Stripe Connect]          │
       │         │                                                              │
       │         ├──> [UI-TRAB-07: Trabajo Asignado & Registrar Llegada]         │
       │         │         │                                                     │
       │         │         ├──> [UI-TRAB-08: Chat en Tiempo Real]                 │
       │         │         │                                                     │
       │         │         └──> [UI-TRAB-09: Finalizar Trabajo (Código/Evid)]   │
       │         │                   │                                           │
       │         │                   └──> [UI-TRAB-11: Evaluación Empleador] ────┤
       │         │                                                               │
       │         ├──> [UI-TRAB-10: Centro de Casos y Soporte]                    │
       │         │                                                               │
       │         ├──> [UI-TRAB-13: Historial Trabajos y Pagos]                    │
       │         │                                                               │
       │         └──> [UI-TRAB-12: Perfil de Trabajador]                        │
```

---

### 4.3 Diagrama de Navegación — Panel Web Administrador

```mermaid
flowchart TD
    UI_ADM_01["UI-ADM-01: Login Admin"] --> UI_ADM_02["UI-ADM-02: Dashboard / Bandeja de Casos"]
    
    UI_ADM_02 --> UI_ADM_03["UI-ADM-03: Detalle y Arbitraje de Reclamaciones"]
    UI_ADM_02 --> UI_ADM_04["UI-ADM-04: Atención de Tickets de Soporte"]
    UI_ADM_02 --> UI_ADM_05["UI-ADM-05: Centro de Moderación de Contenido"]
    UI_ADM_02 --> UI_ADM_06["UI-ADM-06: Gestión y Sanción de Cuentas"]
    UI_ADM_02 --> UI_ADM_07["UI-ADM-07: Monitoreo Financiero y Reintentos Stripe"]
```

```text
[UI-ADM-01: Login Admin]
       │
       └──> [UI-ADM-02: Dashboard / Bandeja de Casos]
                 │
                 ├──> [UI-ADM-03: Detalle y Arbitraje de Reclamaciones]
                 │
                 ├──> [UI-ADM-04: Atención de Tickets de Soporte]
                 │
                 ├──> [UI-ADM-05: Centro de Moderación de Contenido]
                 │
                 ├──> [UI-ADM-06: Gestión y Sanción de Cuentas]
                 │
                 └──> [UI-ADM-07: Monitoreo Financiero y Reintentos Stripe]
```
