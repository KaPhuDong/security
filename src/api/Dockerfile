# Lab 2 buổi chiều: Dockerfile
FROM python:3.13-alpine
RUN pip install flask prometheus-flask-exporter gunicorn \
    && addgroup -S appgroup && adduser -S appuser -G appgroup
COPY app.py /app/app.py
WORKDIR /app
# Chạy với non-root user để pass runAsNonRoot constraint
USER appuser
EXPOSE 8080
# Dùng gunicorn thay flask run (production WSGI server)
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "2", "app:app"]
