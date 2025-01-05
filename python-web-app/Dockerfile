FROM python:3.6-slim

ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1
RUN mkdir /app
WORKDIR /app
RUN pip install --upgrade pip

COPY requirements.txt /app
COPY devops /app

RUN pip install -r requirements.txt && cd devops

EXPOSE 8000

CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]