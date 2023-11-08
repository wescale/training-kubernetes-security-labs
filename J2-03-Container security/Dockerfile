FROM python:3.4-alpine

WORKDIR /app

USER root

ADD requirements.txt /app

RUN pip3 install -r requirements.txt

ADD app.py /app

EXPOSE 8080

CMD ["python", "app.py"]
