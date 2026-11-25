# API de tareas

Aplicación de laboratorio del curso SOFT-756 (Implementación de CI/CD y Prácticas DevOps). Es una
API pequeña en Python (Flask) que guarda tareas en memoria. El código de la aplicación casi no
cambia durante el curso: lo que crece es el **pipeline** que la construye, la prueba, la empaqueta
y la despliega.

## Endpoints

| Método | Ruta | Respuesta |
|---|---|---|
| GET | `/health` | `{"estado": "ok"}` si la API está viva |
| GET | `/version` | Versión desplegada y ambiente (`APP_VERSION`, `APP_ENV`) |
| GET | `/tareas` | Lista de tareas |
| GET | `/tareas/<id>` | Una tarea, o 404 |
| POST | `/tareas` | Crea una tarea con `{"titulo": "..."}`; 201, o 400 si el título no es válido |

Si la variable `API_TOKEN` tiene valor, `POST /tareas` exige el encabezado
`Authorization: Bearer <token>`. Las lecturas no lo exigen.

## Configuración

| Variable | Para qué sirve | Valor por defecto |
|---|---|---|
| `APP_ENV` | Nombre del ambiente (desarrollo, staging, produccion) | `desarrollo` |
| `APP_VERSION` | Versión que reporta `/version` | `dev` |
| `API_TOKEN` | Token que protege `POST /tareas` | sin valor (no se exige) |

## Correr en local

Requisitos: Python 3.10 o posterior, Git y una terminal Bash (en Windows, **Git Bash**).

```bash
python -m venv .venv
source .venv/bin/activate          # en Windows (Git Bash): source .venv/Scripts/activate
python -m pip install -r requirements-dev.txt
bash scripts/pruebas.sh            # corre las pruebas
python -m flask --app app run --port 8000
```

Con la API en marcha, en otra terminal:

```bash
bash scripts/smoke.sh http://localhost:8000
```

Resultado esperado:

```text
Smoke test contra http://localhost:8000
OK /health
OK /version -> {"ambiente":"desarrollo","version":"dev"}
OK /tareas
Smoke test superado
```

## Estructura

```text
app/            código de la API (config.py lee las variables de entorno)
tests/          pruebas con pytest
scripts/        pruebas.sh y smoke.sh: los mismos comandos en local y en el pipeline
docs/           documentación del flujo de entrega
requirements.txt       dependencias de la aplicación, con versión fija
requirements-dev.txt   dependencias para desarrollar y probar
```

## Despliegue y rollback

### Justificación de `.dockerignore`

Docker envía el contexto de construcción al crear la imagen. Estas exclusiones evitan enviar archivos
que no forman parte de la aplicación desplegada, reducen el contexto y ayudan a no incluir datos locales:

| Patrón | Justificación |
|---|---|
| `.git` | Excluye el historial y los metadatos de Git; no se necesitan para ejecutar la API. |
| `.github` | Excluye workflows y configuración de GitHub Actions, que solo sirven para CI/CD. |
| `.venv` | Excluye el entorno virtual local, que es específico de la máquina y no se usa en el contenedor. |
| `venv` | Excluye entornos virtuales locales que usen el nombre `venv`, por la misma razón. |
| `**/__pycache__` | Excluye directorios de caché de Python; se generan al ejecutar el código. |
| `**/*.pyc` | Excluye bytecode de Python que puede regenerarse y no es necesario para construir la imagen. |
| `.pytest_cache` | Excluye la caché generada por pytest; solo es útil durante las pruebas. |
| `.ruff_cache` | Excluye la caché del linter Ruff; no forma parte del runtime. |
| `.coverage` | Excluye el archivo de datos de cobertura generado al ejecutar pruebas. |
| `htmlcov` | Excluye el reporte HTML de cobertura, que no necesita la aplicación. |
| `reportes` | Excluye reportes de pruebas; son artefactos de CI, no dependencias de la API. |
| `tests` | Excluye las pruebas, ya que el contenedor de producción solo necesita la aplicación. |
| `docs` | Excluye documentación que no se requiere para ejecutar la API. |
| `.env` | Excluye configuración local que podría contener secretos; la configuración se inyecta al ejecutar el contenedor. |
