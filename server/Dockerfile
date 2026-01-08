FROM python:3.12-slim

WORKDIR /app

# Copy the entire project first
COPY . .

# Install local mem0 package in editable mode (this installs its dependencies too)
RUN pip install -e .

# Install additional server requirements
RUN pip install --no-cache-dir -r server/requirements.txt

EXPOSE 8080

WORKDIR /app/server

ENV PYTHONUNBUFFERED=1
ENV PYTHONPATH=/app:$PYTHONPATH

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080", "--reload"]
