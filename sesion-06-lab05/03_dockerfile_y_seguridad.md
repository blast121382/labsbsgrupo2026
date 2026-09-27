# LAB-05: Empaquetado en Docker y Seguridad del Contenedor

Este documento analiza la especificación del `Dockerfile` y las reglas de `.dockerignore` diseñadas para garantizar una imagen de despliegue reproducible, segura y ligera.

---

## 1. Definición del `Dockerfile`

```dockerfile
FROM python:3.12-slim
WORKDIR /curso
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY servicio/ servicio/
RUN useradd --create-home docente
USER docente
EXPOSE 8000
CMD ["python", "-m", "uvicorn", "servicio.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## 2. Definición del `.dockerignore`

```dockerignore
*
!Dockerfile
!requirements.txt
!servicio/
!servicio/*.py
```

---

## 3. Principios de Seguridad y Buenas Prácticas Aplicados

### 1. Política de Lista Blanca Estricta (Strict Whitelisting)
El archivo `.dockerignore` inicia con la regla global `*`, ignorando recursivamente cualquier archivo o subdirectorio del repositorio. Posteriormente habilita mediante `!` únicamente lo estrictamente necesario para el funcionamiento en producción:
- **Protección de Secretos:** Excluye rotundamente `.env`, claves API locales, tokens y credenciales de Git.
- **Evita Bloat de Imagen:** Excluye entornos virtuales (`.venv/`), cachés (`__pycache__/`, `.pytest_cache/`), carpetas temporales y salidas intermedias (`salidas/`, `evidencias/`).
- **Aislamiento de Tests:** No incluye la carpeta `tests/` ni `requirements-evaluacion.txt` en la imagen de producción para minimizar la superficie de ataque.

### 2. Ejecución con Usuario Sin Privilegios (Non-Root User)
El comando `RUN useradd --create-home docente` seguido de `USER docente` asegura que el proceso Uvicorn se ejecute bajo un usuario sin privilegios de administrador dentro del contenedor Linux. Esto mitiga riesgos de escape de contenedor (container breakout) ante vulnerabilidades de día cero.

### 3. Eficiencia en la Capa de Caché de Docker (Layer Caching)
Se copia e instala primero `requirements.txt` (`RUN pip install --no-cache-dir -r requirements.txt`) antes de copiar el código fuente `servicio/`. Esto permite que cambios frecuentes en el código de los endpoints no invaliden la caché de instalación de dependencias, reduciendo el tiempo de build de minutos a segundos.

---

## 4. Instrucciones de Despliegue en Entornos con Docker Daemon Activo

```bash
# 1. Construir la imagen local
docker build -t planificador-estrategias-s6 .

# 2. Ejecutar el contenedor inyectando el .env en tiempo de ejecución
docker run --rm --name planificador-estrategias-s6 -p 127.0.0.1:8000:8000 --env-file .env planificador-estrategias-s6

# 3. Comprobar salud del servicio desde otra terminal
curl -fsS http://127.0.0.1:8000/salud
```
