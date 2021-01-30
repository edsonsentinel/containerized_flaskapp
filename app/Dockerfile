FROM python:3.6.1-alpine
COPY src/ /app/
WORKDIR /app
RUN pip install -r requirements.txt
ENTRYPOINT [ "python" ]
CMD [ "server.py" ]

