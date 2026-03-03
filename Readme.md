# dbops-project
Исходный репозиторий для выполнения проекта дисциплины "DBOps"

## Create db `store`

```sh
psql "host=<host> port=<port> dbname=store_default user=<user>"
```

```postgresql
CREATE DATABASE store;
\c store
```

## Create new user
```postgresql
CREATE USER test_user WITH PASSWORD 'test_password';
GRANT ALL PRIVILEGES ON DATABASE store TO test_user;
GRANT USAGE ON SCHEMA public TO test_user;
GRANT CREATE ON SCHEMA public TO test_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL PRIVILEGES ON TABLES TO test_user;
```
