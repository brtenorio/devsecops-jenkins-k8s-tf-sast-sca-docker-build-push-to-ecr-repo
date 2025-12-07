FROM python:3.11-slim

WORKDIR /app

# install minimal build deps if some packages need compilation
RUN apt-get update && apt-get install -y --no-install-recommends gcc build-essential \
    && rm -rf /var/lib/apt/lists/*

# copy requirements from the build context (context is flask-app/)
COPY requirements.txt /app/requirements.txt
RUN pip install --no-cache-dir -r /app/requirements.txt

COPY . /app

ENV PYTHONUNBUFFERED=1
EXPOSE 5000

# adjust the WSGI module/path if your app entrypoint differs (e.g. app:app)
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app", "--workers", "3"]
