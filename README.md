# MercadoTico 360 · Solución orientada a documentos con Apache CouchDB

**Curso:** XS0131 – Gestión de Bases de Datos y Análisis de Información
**Actividad:** Investigación Grupal 1 – Desafío NoSQL
**Caso asignado:** Caso 2 – MercadoTico 360 (modelo de **documentos**)
**Tecnología:** Apache CouchDB 3.x

---

## 1. Descripción

MercadoTico 360 es un marketplace ficticio que agrupa comercios costarricenses. Su catálogo mezcla productos de naturaleza muy distinta (alimentos, ropa, tecnología, artesanías, entre otros), por lo que cada categoría necesita atributos diferentes. En un modelo relacional esto obliga a crear muchas tablas y columnas opcionales.

Este repositorio implementa una alternativa basada en **documentos JSON** sobre Apache CouchDB. Permite:

- almacenar un catálogo heterogéneo sin forzar un esquema idéntico para todos los productos;
- guardar pedidos con estructuras anidadas (líneas de detalle) que conservan lo que el cliente compró aunque el producto cambie después;
- consultar, agregar y evolucionar el esquema sin migraciones relacionales.

## 2. Contenido del repositorio

```
.
├── Simulacion.ipynb               # Genera los datos sintéticos y los carga en CouchDB
├── Creacion_de_requisitos.ipynb   # Ejecuta y demuestra los 7 requisitos del caso
├── docker-compose.yml             # Levanta CouchDB localmente (ver sección 5.2)
├── requirements.txt               # Dependencias de Python (ver sección 5.3)
├── .env.example                   # Plantilla de configuración, sin secretos
├── .gitignore                     # Debe incluir la línea ".env"
└── README.md
```

Los archivos `docker-compose.yml`, `requirements.txt` y `.env.example` se muestran completos en la sección 5, para que pueda crearlos copiando su contenido.

## 3. Requisitos del caso y dónde se cumplen

| N.º | Requisito del caso | Dónde se cumple |
|---|---|---|
| 1 | Diseñar documentos y justificar qué se embebe y qué se referencia | `Simulacion.ipynb` (tabla de diseño) y sección 4 de este documento |
| 2 | Atributos diferentes por categoría | `Simulacion.ipynb` (diccionario `CATALOGO`) |
| 3 | Índices para al menos tres patrones de consulta | `Creacion_de_requisitos.ipynb` (sección *Índices*) |
| 4 | Búsqueda por categoría, rango de precio y atributo específico | `Creacion_de_requisitos.ipynb` (sección *Búsquedas*) |
| 5 | Recuperar pedidos de un cliente y actualizar su estado | `Creacion_de_requisitos.ipynb` (sección *Pedidos*) |
| 6 | Al menos dos agregaciones | `Creacion_de_requisitos.ipynb` (sección *MapReduce*, tres vistas) |
| 7 | Evolución del esquema sin migración | `Creacion_de_requisitos.ipynb` (sección *Evolución*) |

**Escala de los datos generados:** 25 000 productos en 8 categorías, 10 000 clientes y 50 000 pedidos con 1 a 4 líneas de detalle (85 000 documentos en total).

## 4. Modelo de datos

La base de datos usa **un solo contenedor lógico** con tres tipos de documento, distinguidos por el campo `tipo_doc`.

| Tipo | `_id` | Decisión de diseño | Justificación |
|---|---|---|---|
| `producto` | `PROD-000001` | Documento propio con `atributos` **embebidos** | Cada categoría tiene atributos distintos; viven dentro del producto sin esquema fijo. |
| `cliente` | `CLI-000001` | Documento propio | Un cliente puede tener muchos pedidos. Embeberlos haría crecer el documento sin límite. |
| `pedido` | `PED-000001` | `lineas_detalle` **embebidas**; `cliente_id` y `producto_id` **referenciados** | Un pedido casi siempre se lee completo. Cada línea guarda un *snapshot* (nombre, categoría y precio) para reconstruir la compra aunque el producto cambie. |

**Ejemplo de producto** (categoría `Tecnologia`):

```json
{
  "_id": "PROD-000123",
  "tipo_doc": "producto",
  "producto_id": "PROD-000123",
  "nombre": "Laptop 123",
  "categoria": "Tecnologia",
  "precio": 845.5,
  "stock": 120,
  "atributos": { "marca": "TechBrand-7", "ram_gb": 16, "almacenamiento_gb": 512 }
}
```

**Ejemplo de pedido** (estructura anidada):

```json
{
  "_id": "PED-000001",
  "tipo_doc": "pedido",
  "pedido_id": "PED-000001",
  "cliente_id": "CLI-004637",
  "fecha_pedido": "2026-07-14",
  "estado": "pendiente",
  "monto_total": 1253.0,
  "lineas_detalle": [
    {
      "producto_id": "PROD-009498",
      "nombre_snapshot": "Alimento para mascota 9498",
      "categoria_snapshot": "Mascotas",
      "cantidad": 2,
      "precio_unitario": 626.5,
      "subtotal": 1253.0
    }
  ]
}
```

**Índices creados**

| Índice | Campos | Consulta que acelera |
|---|---|---|
| `idx_product_category` | `tipo_doc`, `categoria` | Productos de una categoría |
| `idx_product_price` | `tipo_doc`, `precio` | Productos en un rango de precios |
| `idx_technology_ram` | `tipo_doc`, `categoria`, `atributos.ram_gb` | Atributo específico de una categoría |
| `idx_pedidos_cliente` | `tipo_doc`, `cliente_id` | Historial de pedidos de un cliente |

## 5. Instalación

### 5.1 Prerrequisitos

| Herramienta | Versión sugerida | Para qué se usa |
|---|---|---|
| Python | 3.10 o superior | Ejecutar los notebooks |
| Docker y Docker Compose | Versión reciente | Levantar CouchDB sin instalarlo en el sistema |
| Git | Cualquiera | Clonar el repositorio |

Si ya tiene un servidor CouchDB 3.x en funcionamiento, puede omitir Docker y usar sus propios datos de conexión en la sección 5.4.

### 5.2 Clonar el repositorio y preparar CouchDB

```bash
git clone [URL_DEL_REPOSITORIO]
cd [NOMBRE_DEL_REPOSITORIO]
```

Cree el archivo `docker-compose.yml` con este contenido:

```yaml
services:
  couchdb:
    image: couchdb:3
    container_name: mercadotico-couchdb
    ports:
      - "5984:5984"
    environment:
      COUCHDB_USER: ${COUCHDB_USER}
      COUCHDB_PASSWORD: ${COUCHDB_PASS}
    volumes:
      - couchdb_data:/opt/couchdb/data

volumes:
  couchdb_data:
```

### 5.3 Entorno de Python

Cree el archivo `requirements.txt`:

```text
requests
python-dotenv
notebook
```

Cree y active un entorno virtual, e instale las dependencias:

```bash
python -m venv .venv

# Linux / macOS
source .venv/bin/activate
# Windows (PowerShell)
.venv\Scripts\Activate.ps1

pip install -r requirements.txt
```

### 5.4 Configuración de credenciales

El repositorio **no contiene contraseñas**. Copie la plantilla y complete sus propios valores:

```bash
cp .env.example .env        # En Windows: copy .env.example .env
```

Contenido de `.env.example`:

```text
COUCHDB_HOST=localhost
COUCHDB_PORT=5984
COUCHDB_USER=admin
COUCHDB_PASS=
DATABASE_NAME=mercadotico360
```

Edite `.env` y escriba una contraseña en `COUCHDB_PASS`. Verifique que `.gitignore` contenga la línea `.env` para no subir sus credenciales por accidente.

> Si no existe un archivo `.env`, los notebooks preguntan cada dato al ejecutarse (la contraseña se oculta al escribirla).

### 5.5 Iniciar CouchDB

```bash
docker compose up -d
```

Para comprobar que el servicio responde:

```bash
curl http://localhost:5984/
```

Debe devolver un JSON de bienvenida con la versión de CouchDB. Además, puede explorar los datos desde el panel web **Fauxton**: <http://localhost:5984/_utils>.

## 6. Ejecución

Inicie Jupyter **desde la carpeta raíz del repositorio**, para que se encuentre el archivo `.env`:

```bash
jupyter notebook
```

Ejecute los notebooks **en este orden**, corriendo todas las celdas de arriba hacia abajo.

### Paso 1 · `Simulacion.ipynb` (carga de datos)

Crea la base de datos si no existe, genera los datos con una semilla fija (`SEED = 42`, por lo que los resultados son reproducibles) y los carga por lotes.

**Resultado esperado al final:**

```
Documentos en la BD: 85000 (esperados de esta simulación: 85000)
```

También se imprimen tres documentos de ejemplo (un producto, un cliente y un pedido).

La carga es **idempotente**: cada documento tiene un `_id` propio, así que ejecutarla de nuevo no duplica datos, solo informa cuántos ya existían.

### Paso 2 · `Creacion_de_requisitos.ipynb` (requisitos del caso)

Ejecuta una sección por requisito y, al final, imprime un resumen de cumplimiento:

```
=== RESUMEN DE REQUISITOS · MERCADOTICO 360 ===
[OK] Req 1-2: diseño y atributos variables
[OK] Req 3: índices
[OK] Req 4: antes/después de índices
[OK] Req 4: categoría + precio + atributo
[OK] Req 5: pedidos y actualización
[OK] Req 6: agregaciones MapReduce
[OK] Req 7: evolución del esquema
7/7 requisitos verificados
```

Los tiempos de ejecución dependen del equipo. La creación de índices y la primera consulta a las vistas MapReduce pueden tardar algunos segundos o minutos, porque CouchDB construye esas estructuras sobre los 85 000 documentos.

## 7. Pruebas y evidencias

| Evidencia solicitada | Cómo se obtiene |
|---|---|
| Visualización de documentos reales | Sección *Documentos reales* del segundo notebook, o Fauxton |
| Pruebas antes y después de crear índices | La función `comparar_antes_despues()` elimina los índices de productos, mide una consulta, los recrea y la mide otra vez. Muestra documentos examinados y tiempo (`execution_stats`) |
| Consulta de un pedido anidado y de un producto con atributos variables | Sección *Documentos reales* |
| Evolución del esquema y validación | Sección *Evolución*: categoría nueva, atributo nuevo y rechazo de un producto con precio negativo |

## 8. Efectos que el segundo notebook deja en la base de datos

Conviene conocerlos antes de ejecutarlo:

- Crea tres *design documents*: `_design/pedidos` (cambio de estado), `_design/agregaciones` (vistas MapReduce) y `_design/validacion` (reglas de consistencia).
- Cambia el estado de un pedido del cliente `CLI-000001`.
- Inserta el producto `PROD-999999` (categoría nueva `Tecnologia_Agricola`) y agrega el atributo `certificacion_ecologica` a un producto de la categoría `Ropa`.
- Elimina y vuelve a crear los índices de productos durante la prueba de rendimiento.

La regla de validación rechaza cualquier documento sin un `tipo_doc` válido (`producto`, `cliente` o `pedido`), por lo que otros scripts que escriban en la misma base deben respetarla.

## 9. Reiniciar desde cero

Para borrar todos los datos y empezar de nuevo:

```bash
docker compose down -v      # elimina el contenedor y su volumen de datos
docker compose up -d
```

Luego vuelva a ejecutar los dos notebooks en orden.

## 10. Solución de problemas

| Síntoma | Causa probable | Solución |
|---|---|---|
| `No hay conexión con CouchDB` | El contenedor no está activo o el puerto es otro | Ejecute `docker compose ps` y revise `COUCHDB_PORT` |
| `Usuario o contraseña incorrectos` | La contraseña del `.env` no coincide con la usada al crear el contenedor | Ejecute `docker compose down -v`, corrija `.env` y levante de nuevo |
| `No hay documentos 'producto'` | Se ejecutó el segundo notebook antes que el primero | Ejecute primero `Simulacion.ipynb` |
| Las ventas por categoría aparecen vacías o con clave `null` | La base se cargó con una versión anterior de los datos, sin `categoria_snapshot` | Reinicie desde cero (sección 9) y cargue de nuevo |
| Los notebooks piden los datos de conexión a mano | Jupyter no encontró `.env` | Inicie Jupyter desde la raíz del repositorio |

## 11. Limitaciones

- **Un índice por consulta:** el motor de consultas Mango usa un solo índice por búsqueda; los demás filtros se aplican en memoria sobre los resultados del índice elegido.
- **Actualización de documentos:** CouchDB guarda una nueva revisión completa del documento ante cada cambio. El *update handler* evita que el cliente reenvíe el pedido entero, pero el servidor sí escribe una revisión nueva.
- **Datos sintéticos:** se generan con una distribución uniforme, por lo que no reflejan patrones reales de compra.
- **Instalación de un solo nodo:** la configuración con Docker no incluye réplica ni clúster, por lo que no demuestra la alta disponibilidad de CouchDB.

## 12. Plan de contingencia para la demostración

- Los datos persisten en un volumen de Docker, de modo que reiniciar el contenedor no los pierde.
- Se recomienda cargar los datos y ejecutar ambos notebooks completos antes de la presentación, y conservar capturas de pantalla o un video corto de la ejecución.

## 13. Integrantes

| Nombre | Carnet |
|---|---|---|
| [Carlos Armando Soto Monge] | [Carnet] | 
| [Ericka Marisol Quesada Madrigal] | [Carnet] | 
| [Evelio De Los Ángeles Chinchilla Rosales] | [Carnet] |
| [Gustavo Jhosua Vargas Viales] | [Carnet] |
| [Rodrigo Javier Gutiérrez Calderón] | [Carnet] |

## 14. Referencias

- Apache Software Foundation. *Apache CouchDB Documentation*. <https://docs.couchdb.org/en/stable/>
- Apache Software Foundation. *Mango: consultas declarativas (`_find`) e índices*. <https://docs.couchdb.org/en/stable/api/database/find.html>
- Apache Software Foundation. *Design Documents: vistas, update handlers y validación*. <https://docs.couchdb.org/en/stable/ddocs/ddocs.html>
- ICEX España Exportación e Inversiones, "Informe e-País: Comercio electrónico en Costa Rica 2025 — Resumen ejecutivo," Oficina Económica y Comercial de España en San José, 2025. [En línea]. Disponible: https://www.icex.es/content/dam/icex/centros/costa-rica/documentos/2025/informe-epais-comercio-electronico-costa-rica-2025-resumen-ejecutivo.pdf
- NIC Costa Rica, "NIC Costa Rica participó en el Ecommerce & Delivery Forum para acercar el comercio en línea a las PYMES," 2 sep. 2026. [En línea]. Disponible: https://nic.cr/2026/09/02/nic-costa-rica-participo-en-el-ecommerce-delivery-forum-para-acercar-el-comercio-en-linea-a-las-pymes/
- E. F. Codd, "A relational model of data for large shared data banks," Commun. ACM, vol. 13, no. 6, pp. 377–387, jun. 1970, doi: 10.1145/362384.362685.
- R. Agrawal, A. Somani y Y. Xu, "Storage and querying of e-commerce data," en Proc. 27th Int. Conf. Very Large Data Bases (VLDB), Roma, Italia, 2001, pp. 149–158.
- R. Cattell, "Scalable SQL and NoSQL data stores," ACM SIGMOD Record, vol. 39, no. 4, pp. 12–27, 2011, doi: 10.1145/1978915.1978919.
- M. Stonebraker, "SQL databases v. NoSQL databases," Commun. ACM, vol. 53, no. 4, pp. 10–11, abr. 2010, doi: 10.1145/1721654.1721659.
- F. Gessert, W. Wingerath, S. Friedrich y N. Ritter, "NoSQL database systems: A survey and decision guidance," Comput. Sci. Res. Dev., vol. 32, no. 3–4, pp. 353–365, 2017, doi: 10.1007/s00450-016-0334-3.