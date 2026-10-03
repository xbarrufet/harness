# Preparación del entorno de desarrollo

Esta guía describe las actividades para preparar un entorno local para desarrollar el harness. El proyecto está todavía en fase de diseño; por eso, estos pasos preparan una base de trabajo y no ejecutan aún una aplicación.

## 1. Instalar y comprobar Python

Se recomienda **Python 3.11**, coherente con la versión contemplada en el ejemplo de CI del descriptor. Comprueba que el intérprete esté disponible:

```bash
python3.11 --version
```

Si el comando no existe, instala Python 3.11 desde una distribución de confianza y vuelve a comprobar la versión. En Windows, usa `py -3.11` en lugar de `python3.11` en los comandos siguientes.

## 2. Crear el entorno virtual

Desde el directorio raíz del repositorio, crea un entorno virtual local llamado `.venv`:

```bash
python3.11 -m venv .venv
```

En Windows PowerShell:

```powershell
py -3.11 -m venv .venv
```

El entorno virtual mantiene las dependencias del proyecto separadas de las instalaciones globales de Python.

## 3. Activar el entorno

En macOS o Linux:

```bash
source .venv/bin/activate
```

En Windows PowerShell:

```powershell
.venv\Scripts\Activate.ps1
```

Una vez activado, la terminal suele mostrar `(.venv)` en el prompt. Confirma que se está usando el intérprete del entorno:

```bash
python --version
python -m pip --version
```

Para salir del entorno cuando termines:

```bash
deactivate
```

## 4. Actualizar las herramientas de instalación

Con el entorno activado, actualiza `pip`:

```bash
python -m pip install --upgrade pip
```

## 5. Instalar dependencias iniciales

Mientras no exista un manifiesto de dependencias del proyecto, instala las librerías contempladas en el diseño directamente en el entorno virtual:

```bash
python -m pip install "langgraph>=0.2.0" "pydantic>=2" python-dotenv langchain-openai
```

Instala las herramientas previstas para desarrollo y validación:

```bash
python -m pip install pytest pytest-json-report ruff mypy bandit
```

`langchain-openai` es una opción de integración con proveedor LLM mencionada en el descriptor. Si se elige otro proveedor, sustituye o complementa ese paquete con el adaptador correspondiente. Docker, SonarQube y GitHub CLI son herramientas externas opcionales para las fases que las necesiten; no se instalan con `pip`.

> Estas dependencias todavía no están fijadas en archivos de requisitos. Antes de compartir el entorno o incorporarlo a CI, se deberán definir versiones compatibles y registrar las dependencias del proyecto en un manifiesto reproducible.

## 6. Proteger los archivos locales

Asegúrate de que `.venv/`, los archivos `.env` y los artefactos temporales no se añadan al control de versiones. Añade estas entradas al `.gitignore` del proyecto si aún no están incluidas:

```gitignore
.venv/
.env
.env.*
!.env.example
__pycache__/
.pytest_cache/
.mypy_cache/
.ruff_cache/
```

Guarda las claves de API únicamente en un archivo `.env` local, nunca en el repositorio. Si se necesita documentar la configuración, crea un `.env.example` sin credenciales reales.

## 7. Verificar las herramientas

Con el entorno activo, comprueba que las herramientas principales se pueden invocar:

```bash
python -m pip check
python -m pytest --version
ruff --version
mypy --version
bandit --version
```

`pip check` debe finalizar sin conflictos de dependencias. La suite de tests todavía no está disponible mientras no se implemente el código.

## 8. Uso cotidiano

Cada vez que abras una terminal nueva en el repositorio:

1. Activa `.venv` con el comando correspondiente a tu sistema operativo.
2. Comprueba que el prompt indica `(.venv)` si tu terminal lo muestra.
3. Ejecuta las herramientas y pruebas desde ese entorno.
4. Antes de cerrar, desactívalo con `deactivate` si quieres volver al Python global.

Para el diseño de componentes, fases y tecnologías del sistema, consulta [descriptor.md](descriptor.md). Esta guía se actualizará cuando se añadan el código de la aplicación y sus manifiestos de dependencias.
