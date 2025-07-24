
CREATE ROLE helloworldspcs_admin;
CREATE ROLE helloworldspcs_user;

CREATE OR REPLACE WAREHOUSE helloworldspcs_warehouse WITH WAREHOUSE_SIZE='X-SMALL';

CREATE COMPUTE POOL helloworldspcs_compute_pool
  MIN_NODES = 1
  MAX_NODES = 5
  INSTANCE_FAMILY = CPU_X64_XS;

create database helloworldspcs_db;
create schema helloworldspcs_schema;

use role helloworldspcs_admin;

create image repository helloworldspcs_db.helloworldspcs_schema.img_repo;

GRANT OWNERSHIP ON image repository helloworldspcs_db.helloworldspcs_schema.img_repo TO ROLE helloworldspcs_admin COPY CURRENT GRANTS;

GRANT OWNERSHIP ON DATABASE helloworldspcs_db TO ROLE helloworldspcs_admin COPY CURRENT GRANTS;
GRANT OWNERSHIP ON schema helloworldspcs_db.helloworldspcs_schema TO ROLE helloworldspcs_admin COPY CURRENT GRANTS;

GRANT USAGE ON WAREHOUSE helloworldspcs_warehouse TO ROLE helloworldspcs_admin;

GRANT USAGE, MONITOR ON COMPUTE POOL helloworldspcs_compute_pool TO ROLE helloworldspcs_admin;
GRANT BIND SERVICE ENDPOINT ON ACCOUNT TO ROLE helloworldspcs_admin;

--------------------------------------
-- SETUP USER ROLE - NOT NECESSARY IF USING ADMIN
--------------------------------------

GRANT USAGE ON DATABASE helloworldspcs_db TO ROLE helloworldspcs_user;
GRANT USAGE ON SCHEMA helloworldspcs_db.helloworldspcs_schema TO ROLE helloworldspcs_user;
GRANT USAGE ON SERVICE helloworldspcs_db.helloworldspcs_schema.helloworldspcs_frontend TO ROLE helloworldspcs_user;
GRANT USAGE ON SERVICE helloworldspcs_db.helloworldspcs_schema.helloworldspcs_backend TO ROLE helloworldspcs_user;

GRANT SERVICE ROLE helloworldspcs_db.helloworldspcs_schema.helloworldspcsfrontend!ALL_ENDPOINTS_USAGE TO ROLE helloworldspcs_user;
GRANT SERVICE ROLE helloworldspcs_db.helloworldspcs_schema.helloworldspcsbackend!ALL_ENDPOINTS_USAGE TO ROLE helloworldspcs_user;

--------------------------------------
-- CREATE SERVICES -- Make sure you push your Docker Containers First! 
--------------------------------------

CREATE SERVICE helloworldspcsfrontend
  IN COMPUTE POOL helloworldspcs_compute_pool
  FROM SPECIFICATION $$
    spec:
      containers:
      - name: helloworld-frontend
        image: /helloworldspcs_db/helloworldspcs_schema/img_repo/helloworld_frontend:latest
        env:
          BACKEND_SERVICE_NAME: "helloworldspcsbackend"
          BACKEND_SERVICE_PORT: 3000
      endpoints:
      - name: helloworld-frontend-endpoint
        port: 3000
        public: true
      $$
   MIN_INSTANCES=1
   MAX_INSTANCES=1;

  

CREATE SERVICE helloworldspcsbackend
  IN COMPUTE POOL helloworldspcs_compute_pool
  FROM SPECIFICATION $$
    spec:
      containers:
      - name: helloworld-backend
        image: /helloworldspcs_db/helloworldspcs_schema/img_repo/helloworld_backend:latest
        env:
          REDIS_SERVICE_NAME: "helloworldspcsredis"
          REDIS_SERVICE_PORT: 6379
          SF_WAREHOUSE: "helloworldspcs_warehouse"
      endpoints:
      - name: helloworld-backend-endpoint
        port: 3000
        public: true
      $$
   MIN_INSTANCES=1
   MAX_INSTANCES=1;

CREATE SERVICE helloworldspcsredis
  IN COMPUTE POOL helloworldspcs_compute_pool
  FROM SPECIFICATION $$
    spec:
      containers:
      - name: helloworld-redis
        image: /helloworldspcs_db/helloworldspcs_schema/img_repo/redis:latest
      endpoints:
      - name: helloworld-redis-endpoint
        port: 6379
        public: true
      $$
   MIN_INSTANCES=1
   MAX_INSTANCES=1;

--------------------------------------
-- HELPFUL COMMANDS
--------------------------------------

SHOW IMAGE REPOSITORIES;

SHOW SERVICES;

show endpoints in service helloworldspcsfrontend;
--This where you will find the ingress endpoints to add to the Caddyfile setup. 

describe service helloworldspcsfrontend;

SELECT SYSTEM$GET_SERVICE_LOGS(
    'helloworldspcs_db.helloworldspcs_schema.helloworldspcsfrontend',
    0,
    'helloworld-frontend'
);

SELECT SYSTEM$GET_SERVICE_LOGS(
    'helloworldspcs_db.helloworldspcs_schema.helloworldspcsbackend',
    0,
    'helloworld-backend'
);

SELECT SYSTEM$GET_SERVICE_LOGS(
    'helloworldspcs_db.helloworldspcs_schema.helloworldspcsredis',
    0,
    'helloworld-redis'
);

--------------------------------------
-- CREATE A TEST TABLE
--------------------------------------

CREATE OR REPLACE TABLE MY_TEST_TABLE (
    ID INT,
    NAME VARCHAR(50),
    VALUE DECIMAL(10, 2),
    CREATED_AT TIMESTAMP_NTZ
);

-- Insert 10 test rows
INSERT INTO MY_TEST_TABLE (ID, NAME, VALUE, CREATED_AT) VALUES
    (1, 'Alpha', 100.50, '2023-01-01 10:00:00'),
    (2, 'Beta', 150.75, '2023-01-02 11:30:00'),
    (3, 'Gamma', 200.00, '2023-01-03 12:45:00'),
    (4, 'Delta', 75.20, '2023-01-04 14:00:00'),
    (5, 'Epsilon', 300.99, '2023-01-05 15:15:00'),
    (6, 'Zeta', 120.00, '2023-01-06 16:30:00'),
    (7, 'Eta', 50.10, '2023-01-07 17:45:00'),
    (8, 'Theta', 250.60, '2023-01-08 18:00:00'),
    (9, 'Iota', 180.30, '2023-01-09 19:15:00'),
    (10, 'Kappa', 90.45, '2023-01-10 20:30:00');

GRANT OWNERSHIP ON table helloworldspcs_db.helloworldspcs_schema.MY_TEST_TABLE TO ROLE helloworldspcs_admin COPY CURRENT GRANTS;

--------------------------------------
-- GENERATE PAT TOKEN FOR SERVICE USER
--------------------------------------

use role helloworldspcs_admin;

CREATE USER helloworldspcs_service_acct
  PASSWORD = 'StrongPassword!123'
  DEFAULT_ROLE = helloworldspcs_admin
  DEFAULT_WAREHOUSE = helloworldspcs_warehouse
  DEFAULT_NAMESPACE = helloworldspcs_db.helloworldspcs_schema
  MUST_CHANGE_PASSWORD = TRUE
  COMMENT = 'Service user for app access';

grant role helloworldspcs_admin to user helloworldspcs_service_acct; --just in case

CREATE AUTHENTICATION POLICY helloworldspcs_pat_policy
 PAT_POLICY = (
    NETWORK_POLICY_EVALUATION = NOT_ENFORCED
);
ALTER USER helloworldspcs_service_acct SET AUTHENTICATION POLICY helloworldspcs_pat_policy;
GRANT MODIFY PROGRAMMATIC AUTHENTICATION METHODS ON USER helloworldspcs_service_acct
TO ROLE helloworldspcs_admin;
ALTER USER IF EXISTS helloworldspcs_service_acct ADD PROGRAMMATIC ACCESS TOKEN mdw_token
COMMENT = 'mdw pat token';




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
