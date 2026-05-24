FROM python:3.12-slim

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

WORKDIR /seed

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY seed_all.py .

CMD ["python", "seed_all.py"]
