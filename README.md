# Base_de_conocimientos
levantamiento de podman aplicatin complete
# Knowledge Base de Repositorios

App sencilla para guardar y organizar tus repositorios (nombre, URL, descripción,
lenguaje, tags, notas, favoritos) con búsqueda y estadísticas. Backend en FastAPI,
base de datos en PostgreSQL, UI servida desde el mismo backend (sin frontend
separado que compilar).

Arquitectura: **2 pods**
- `kb-db-pod` → PostgreSQL 16 con volumen persistente
- `kb-app-pod` → API + UI (FastAPI/uvicorn)

---

## Opción A — Rápida, con podman-compose (recomendada)

```bash
# 1. Instalar podman-compose si no lo tienes
pip install --user podman-compose
# o: sudo dnf install podman-compose   /   sudo apt install podman-compose

# 2. Copiar variables de entorno (opcional, trae defaults funcionales)
cp .env.example .env

# 3. Levantar todo
podman-compose up -d --build

# 4. Ver logs
podman-compose logs -f

# 5. Apagar
podman-compose down          # conserva datos
podman-compose down -v       # borra también el volumen de datos
```

Abre `http://<ip-del-servidor>:8000`

---

## Opción B — Pods nativos de Podman (sin compose)

Usa `podman pod create` + `podman run` directamente, resolviendo la IP entre
pods automáticamente. Útil si tu servidor no tiene `podman-compose` instalado.

```bash
chmod +x build_app.sh start_pods.sh stop_pods.sh   # ya vienen con permiso

./start_pods.sh     # construye la imagen, crea ambos pods, conecta la DB
```

Esto:
1. Construye la imagen `localhost/kb-app:latest`
2. Crea `kb-db-pod` con PostgreSQL (puerto 5432) y volumen `kb_db_data`
3. Espera a que la base esté lista (`pg_isready`)
4. Detecta automáticamente la IP del contenedor de base de datos
5. Crea `kb-app-pod` con la app apuntando a esa IP (puerto 8000)

Para apagar:
```bash
./stop_pods.sh
```

### Alternativa: `podman play kube`
También incluyo `kube.yml` por si prefieres el enfoque declarativo
estilo Kubernetes (`podman play kube kube.yml`). Antes de usarlo:
1. Corre `./build_app.sh` para tener la imagen lista.
2. Si usas esta vía, Podman **no resuelve DNS entre pods separados**
   automáticamente — tendrás que editar manualmente el campo `DB_HOST`
   en `kube.yml` con la IP real del pod de base de datos una vez creado
   (`podman inspect kb-db-pod`). Por eso el script `start_pods.sh` es la
   vía recomendada: ya lo resuelve por ti.

---

## Estructura del proyecto

```
knowledge-repo/
├── app/
│   ├── main.py            # API REST + ruta de la UI
│   ├── db.py               # pool de conexión a Postgres (asyncpg)
│   ├── requirements.txt
│   ├── Dockerfile
│   ├── templates/index.html
│   └── static/{style.css, app.js}
├── db-init/init.sql        # esquema inicial, se ejecuta solo al crear la DB
├── podman-compose.yml       # opción A
├── kube.yml                 # opción declarativa alternativa
├── build_app.sh / start_pods.sh / stop_pods.sh   # opción B
└── .env.example
```

## Endpoints principales

| Método | Ruta                       | Descripción                          |
|--------|----------------------------|---------------------------------------|
| GET    | `/`                         | UI web                               |
| GET    | `/api/repositories`         | Listar (filtros: `q`, `language`, `favorite`) |
| POST   | `/api/repositories`         | Crear repositorio                    |
| GET    | `/api/repositories/{id}`    | Detalle                              |
| PUT    | `/api/repositories/{id}`    | Editar (parcial)                     |
| DELETE | `/api/repositories/{id}`    | Eliminar                             |
| GET    | `/api/stats`                | Totales y conteo por lenguaje        |
| GET    | `/api/health`               | Healthcheck                          |
| GET    | `/docs`                     | Swagger interactivo (autogenerado)   |

## Notas

- Los datos del Postgres viven en el volumen `kb_db_data` — sobreviven a
  reinicios y a recrear el contenedor de la app.
- `init.sql` solo corre la primera vez que se crea el volumen de datos vacío
  (comportamiento estándar de la imagen oficial de Postgres).
- Si cambias el puerto 8000 o 5432 porque ya los usas en tu servidor, ajústalo
  en `podman-compose.yml` o en `start_pods.sh`.
- La UI no requiere build step (no hay React/Vue): es HTML+CSS+JS plano servido
  directo por FastAPI, así que el contenedor de la app es liviano y rápido de
  levantar.
