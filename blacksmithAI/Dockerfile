FROM python:3.12-slim

WORKDIR /app

# Install system dependencies (minimal)
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copy dependency files first (better layer caching)
COPY pyproject.toml ./

# Install Python dependencies with optimized settings
RUN pip install --no-cache-dir --compile uv \
    && uv pip install --system --no-cache-dir -e .

# Copy application code
COPY . .

# Set production environment variables
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONHASHSEED=random \
    LANGCHAIN_TRACING_V2=false \
    OTEL_SDK_DISABLED=true

EXPOSE 8000

# Use production Uvicorn with optimized workers
CMD ["uvicorn", "main:create_app", "--host", "0.0.0.0", "--port", "8000", "--workers", "1", "--limit-concurrency", "10", "--backlog", "20", "--timeout-keep-alive", "5"]
