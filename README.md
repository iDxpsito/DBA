# VitalPet Veterinaria S.A.S. — Proyecto Final de Bases de Datos Avanzadas

> Sistema de gestión clínica veterinaria sobre Oracle Live SQL.
> Archivo principal evaluable: **`proyecto_final_vitalpet_[apellido1]_[apellido2].sql`**

---

## 1. Portada y contexto corporativo

**Empresa ficticia:** VitalPet Veterinaria S.A.S.

**Misión y visión del sistema:** VitalPet resuelve la gestión clínica y de inventario de una red de clínicas veterinarias. La base de datos centraliza dueños, mascotas, veterinarios, consultas y el inventario de medicamentos, garantizando que ninguna fórmula médica se registre sin existencias reales y que los datos comerciales sensibles (costo de adquisición de medicamentos y contacto personal de los clientes) queden protegidos. El objetivo es soportar la operación diaria de varias sedes en simultáneo sin inconsistencias de inventario.

**Integrantes del equipo:**

| Nombre completo | Código | Rol simulado |
|---|---|---|
| [Tu nombre] | [código] | Arquitecto de Datos |
| Eric David Reyes Benítez | [código] | Desarrollador PL/SQL |
| Brayan Cárdenas Rodríguez | [código] | Líder de QA |
| Fabio Nicolás González Plazas | [código] | Analista de Reportería |

---

## 2. Lógica de negocio y requerimientos

### 2.1 Reglas de negocio

1. **Regla de precios:** ningún medicamento ni tarifa de veterinario puede registrarse con un valor menor o igual a cero.
2. **Regla de inventario:** no se puede recetar una cantidad de medicamento superior a las existencias físicas en bodega.
3. **Regla de seguridad financiera:** recepción y clientes externos tienen prohibido visualizar el costo de adquisición interno de los medicamentos y el contacto personal de los dueños.
4. **Regla de concurrencia:** dos atenciones simultáneas no pueden descontar el mismo medicamento al punto de dejar el inventario en negativo.

### 2.2 Mapeo de requerimientos vs. objetos SQL

| Requerimiento del negocio | Componente técnico | Objeto en el código |
|---|---|---|
| Prevenir inventario en negativo | CONSTRAINT CHECK / TRIGGER | `chk_stock_no_negativo` / `trg_validar_stock_medicamento` |
| Evitar precios/tarifas inválidos | CONSTRAINT CHECK | `chk_precio_venta_positivo` / `chk_tarifa_positiva` |
| Evitar bloqueos infinitos entre sedes | Control de concurrencia ACID | Bloque PL/SQL con `FOR UPDATE WAIT 3` |
| Fallar de inmediato si el recurso está ocupado | Control de concurrencia ACID | Bloque PL/SQL con `FOR UPDATE NOWAIT` |
| Ocultar costos internos y contacto del dueño | Vista de seguridad | `v_catalogo_medicamentos` |
| Registrar una atención completa de forma atómica | Stored Procedure | `registrar_atencion` |
| Saber qué veterinario genera más ingresos | Vista de reporte | `v_ingresos_por_veterinario` |
| Auditar cambios de inventario | Trigger de auditoría | `trg_auditar_stock` → `log_auditoria` |

---

## 3. Modelado

### 3.1 Modelo relacional (cardinalidad 1:N)

```
DUENOS (1) ──< (N) MASCOTAS (1) ──< (N) CONSULTAS (N) >── (1) VETERINARIOS
                                          (1)
                                           │
                                           ∧
                                          (N)
                              MEDICAMENTOS_CONSULTA (N) >── (1) MEDICAMENTOS
                                                                     │ (UPDATE stock)
                                                                     ∧
                                                              LOG_AUDITORIA
```

> Reemplaza este bloque por una imagen real del DER en `docs/diagrama_ER.png` (con PK, FK y la tabla de auditoría visibles).

### 3.2 Diccionario de datos (ejemplo: `medicamentos`)

| Columna | Tipo | Restricción / Descripción |
|---|---|---|
| `medicamento_id` | NUMBER | PK, autogenerada por identidad |
| `nombre` | VARCHAR2(100) | No nulo, nombre comercial |
| `categoria` | VARCHAR2(40) | No nulo |
| `stock` | NUMBER | No nulo, `CHECK (stock >= 0)` |
| `precio_venta` | NUMBER(10,2) | No nulo, `CHECK (precio_venta > 0)` |
| `costo_interno` | NUMBER(10,2) | No nulo, **dato sensible (no se expone)** |
| `stock_minimo` | NUMBER | `DEFAULT 5` |

---

## 4. Justificación de la arquitectura transaccional (ACID)

**Análisis de concurrencia:** el peor escenario es que dos veterinarios, en dos sedes distintas, formulen al mismo milisegundo el último frasco de un medicamento. Sin bloqueo, ambos leerían el mismo stock, ambos descontarían y el inventario quedaría en negativo. Con `SELECT ... FOR UPDATE` la fila se bloquea desde la lectura, así que el segundo proceso espera o falla de forma controlada.

**Justificación del `WAIT 3`:** se eligieron 3 segundos porque es el margen de tolerancia del backend antes de que la aplicación web dispare un timeout al usuario. Más de 3 s daría una sensación de aplicación congelada; menos no daría tiempo a que la otra transacción confirme.

**Uso de excepciones especializadas:** se capturan los códigos nativos `-54` (`ORA-00054`) y `-30006` (`ORA-30006`) con `PRAGMA EXCEPTION_INIT` en lugar de un `WHEN OTHERS` genérico, porque así el código distingue exactamente *qué* falló (recurso ocupado vs. timeout) y reacciona con el mensaje correcto, sin silenciar errores graves de otra naturaleza.

---

## 5. Diseño de seguridad y reportería

**Estrategia de seguridad (`v_catalogo_medicamentos`):** se eliminaron `costo_interno` (revela el margen comercial) y `stock_minimo` (dato logístico interno). El stock exacto se reemplaza por un estado calculado con `CASE` (`Disponible` / `Pocas unidades` / `Agotado`), de modo que recepción consulta el catálogo sin acceder a la tabla base.

**Pregunta de negocio resuelta (`v_ingresos_por_veterinario`):** *"¿Qué veterinario genera más ingresos por fórmulas médicas, cuántas consultas atiende y cuál es el promedio de unidades por atención?"* — información directa para la junta directiva sobre productividad por profesional.

---

## 6. Conclusiones y buenas prácticas encontradas

- Delegar la validación de stock al servidor (TRIGGER) garantiza la regla aunque alguien inserte directo en la tabla, saltándose la aplicación: la integridad vive en la base de datos, no en el cliente.
- El uso de un `TYPE` (`t_consulta`) permitió encapsular los datos clínicos como una unidad lógica dentro de la consulta.
- La nomenclatura estricta (`fk_`, `chk_`, `trg_`, `v_`) hizo que integrar los bloques de cada integrante en un único `.sql` fuera directo, sin colisiones de nombres.

---

## Instrucciones de ejecución

1. Abrir [Oracle Live SQL](https://livesql.oracle.com).
2. Pegar el contenido de `proyecto_final_vitalpet_[apellido1]_[apellido2].sql`.
3. Ejecutar con **F5 (Run Script)**. El script corre de principio a fin sin intervención manual.
4. Verificar que la última consulta de autoevaluación devuelva **0 filas** (todos los objetos `VALID`).

```
nombre-del-repositorio/
├── README.md
├── proyecto_final_vitalpet_apellido1_apellido2.sql   <- archivo evaluable
└── docs/
    ├── diagrama_ER.png
    └── capturas_ejecucion/
```
