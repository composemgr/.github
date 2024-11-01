## 👋 Welcome to composemgr 🚀  

All of these docker compose files use the same env settings  
create the file "$HOME/.config/myscripts/composemgr/compose.env"  
and edit it with the following variables  

## export global variables

```shell
export COMPOSE_ENV_FILES="$HOME/.config/myscripts/composemgr/compose.env"
```

```shell
mkdir -p "$HOME/.config/myscripts/composemgr"
${EDITOR:-vim} "$HOME/.config/myscripts/composemgr/compose.env"
```

## SYSTEM Name ENV

```shell
BASE_HOST_NAME="$HOSTNAME"
```

## Database ENV

```shell
REDIS_URL="redis"
VALKEY_URL="valkey"
COUCHDB_URL="couchdb"
MARIADB_URL="mariadb"
MONGODB_URL="mongodb"
MSSQLDB_URL="mssqldb"
COUCHBASE_URL="couchbasedb"
POSTGRESQL_URL="postgresqldb"
POCKETBASE_URL="pocketbasedb"
DB_ADMIN_NAME=""
DB_ADMIN_PASS=""
DB_USER_NAME=""
DB_USER_PASS=""
DB_CREATE_DATABASE_NAME=""
```

## USER ENV  

```shell
APP_RUN_AS=""
APP_USER_NAME=""
APP_USER_PASS=""
APP_ROOT_PASS=""
```

## E-Mail ENV

```shell
EMAIL_SERVER_HOST=""
EMAIL_SERVER_PORT=""
EMAIL_SERVER_TIMEOUT="10"
EMAIL_SERVER_FROM_NAME=""
EMAIL_SERVER_FROM_EMAIL=""
EMAIL_SERVER_LOGIN_NAME=""
EMAIL_SERVER_LOGIN_PASS=""
```

## KEYS ENV

```shell
ENCRYPTION_KEY=""
RPC_SECRET=""
```

## Create random password

```shell
tr -dc A-Za-z0-9 </dev/urandom | head -c 20; echo
```

## Create hash password

```shell
openssl passwd -1 -salt $(openssl rand -base64 6) $(tr -dc A-Za-z0-9 <"/dev/urandom" | head -c 20)
```

## Create secret

```shell
openssl rand -hex 32
```

## Author  

🤖 casjay: [Github](https://github.com/casjay) 🤖  
