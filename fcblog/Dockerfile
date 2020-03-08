FROM python:3.6-alpine

RUN adduser -D fcblog

WORKDIR /home/fcblog

COPY requirements.txt requirements.txt
RUN python -m venv venv
RUN venv/bin/pip install -r requirements.txt
RUN venv/bin/pip install gunicorn pymysql

COPY app app
COPY migrations migrations
COPY fcblog.py config.py start.sh ./
RUN chmod a+x start.sh

ENV FLASK_APP fcblog.py

RUN chown -R fcblog:fcblog ./
USER fcblog

EXPOSE 5000
ENTRYPOINT ["./start.sh"]