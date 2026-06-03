# Stage 1: The Builder
FROM ghcr.io/astral-sh/uv:python3.13-bookworm-slim AS builder
ENV UV_HTTP_TIMEOUT=120
WORKDIR /app

# Copy the dependency files first for docker caching
COPY pyproject.toml uv.lock ./

# Copy the specific services files
COPY services/order-service/pyproject.toml services/order-service/

# Install dependencies for the order-service
RUN uv sync --package order-service --frozen --no-dev

#  Copy the rest of the code
COPY . .

# Stage 2: The Final Image
FROM python:3.13-slim-bookworm
WORKDIR /app

# Copy the environment and code from the builder
COPY --from=builder /app /app

# Put the virtual environment on the path
ENV PATH="/app/.venv/bin:$PATH"

ENV PYTHONPATH="/app/services/order-service:/app"

# Run the FastAPI server.
CMD ["fastapi", "run", "services/order-service/main.py", "--port", "8000", "--host", "0.0.0.0"]