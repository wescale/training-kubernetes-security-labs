FROM python:3.12-alpine

WORKDIR /app

USER root

COPY requirements.txt /app

RUN apk add --upgrade openssl && \
    pip3 install --upgrade pip && \
    pip3 install -r requirements.txt && \
    adduser -u 10001 -D myuser && \
    addgroup --gid 20000 mygroup && \
    addgroup myuser mygroup

COPY app.py /app

USER 10001

EXPOSE 8080

HEALTHCHECK CMD curl --fail http://localhost:8080

CMD ["python", "app.py"]
