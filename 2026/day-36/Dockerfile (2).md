FROM python:3.11-alpine

RUN apk add --no-cache postgresql-client && \
    addgroup -g 1000 -S appgroup && \
    adduser -S appuser -u 1000 -G appgroup

WORKDIR /app


COPY app/ .

RUN pip install --no-cache-dir -r requirements.txt

USER appuser
EXPOSE 5000
CMD ["python", "app.py"]
~                             