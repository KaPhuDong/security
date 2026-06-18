# Lab 2 buổi chiều: Dockerfile
FROM python:3.13-alpine
RUN pip install flask prometheus-flask-exporter gunicorn
COPY app.py /app/app.py
WORKDIR /app
EXPOSE 8080
# Dùng gunicorn thay flask run (production WSGI server)
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "2", "app:app"]
