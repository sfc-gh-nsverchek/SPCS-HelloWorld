docker build --rm --platform linux/amd64 -f Dockerfile -t <account id>.registry.snowflakecomputing.com/helloworldspcs_db/helloworldspcs_schema/img_repo/helloworld_frontend:latest .
docker push <account id>.registry.snowflakecomputing.com/helloworldspcs_db/helloworldspcs_schema/img_repo/helloworld_frontend:latest
```

```
docker build --rm --platform linux/amd64 -f Dockerfile -t <account id>.registry.snowflakecomputing.com/helloworldspcs_db/helloworldspcs_schema/img_repo/helloworld_backend:latest .
docker push <account id>.registry.snowflakecomputing.com/helloworldspcs_db/helloworldspcs_schema/img_repo/helloworld_backend:latest
```

```
docker pull amd64/redis
docker tag amd64/redis:latest tqpzuaf-osa60413.registry.snowflakecomputing.com/helloworldspcs_db/helloworldspcs_schema/img_repo/redis:latest
docker push <account id>.registry.snowflakecomputing.com/helloworldspcs_db/helloworldspcs_schema/img_repo/redis:latest
```
