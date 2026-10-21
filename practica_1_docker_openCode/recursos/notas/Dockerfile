FROM python:3.12-slim

# curl es necesario para el HEALTHCHECK; se limpia la caché de apt
RUN apt-get update \
    && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=1

WORKDIR /app

# 1) Dependencias ANTES del código para aprovechar la caché de capas
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 2) Código de la aplicación
COPY app.py .
COPY templates ./templates

# 3) Usuario sin privilegios
RUN useradd --create-home --shell /bin/bash appuser \
    && chown -R appuser:appuser /app
USER appuser

EXPOSE 8000

# curl no viene en la imagen base, por eso se instala arriba
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -fsS http://localhost:8000/salud || exit 1

# 4) Arranque con gunicorn (no el servidor de desarrollo de Flask)
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "2", "app:app"]
