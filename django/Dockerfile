FROM python:3


COPY requirements.txt /usr/src/app/

WORKDIR /usr/src/app/

RUN pip install --no-cache-dir -r /usr/src/app/requirements.txt

COPY . /usr/src/app

EXPOSE 8000

CMD ["python3", "manage.py", "runserver"]
