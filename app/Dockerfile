FROM ubuntu:latest

RUN apt-get update && \
    apt-get install -y apache2 zip unzip && \
    rm -rf /var/lib/apt/lists/*

ADD https://www.free-css.com/assets/files/free-css-templates/download/page258/loxury.zip /var/www/html/
WORKDIR /var/www/html
RUN unzip loxury.zip && \
    cp -rvf loxury/* . && \
    rm -rf loxury loxury.zip

CMD ["apache2ctl", "-D", "FOREGROUND"]

EXPOSE 80
