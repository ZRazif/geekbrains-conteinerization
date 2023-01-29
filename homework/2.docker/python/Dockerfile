FROM python:3.9-alpine

ADD homework/2.docker/python /app

RUN pip install -r /app/requirements.txt

EXPOSE 8080

ENTRYPOINT python /app/app.py runserver 0.0.0.0:8080