FROM mysql:5.7
ENV MYSQL_ROOT_PASSWORD=${PASSWORD}
COPY ./Create.sql /docker-entrypoint-initdb.d
