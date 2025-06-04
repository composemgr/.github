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

## SYSTEM VARS

```shell
TZ="America/New_York"
BASE_HOST_NAME="$HOSTNAME"
BASE_DOMAIN_NAME="$(hostname -d 2>/dev/null | grep '^' || echo "$HOSTNAME")"
BASE_STORAGE_DIR="/var/lib/srv/$USER/docker"
BASE_DATABASE_DIR="/var/lib/srv/$USER/databases" 
```

## user and group

```shell
USER_UID=$(id -u)
USER_GID=$(id -g)
```

## DATABASE HOSTNAMES ENV

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
```

## DATABASE USER ENV

```shell
DB_ADMIN_NAME="${DB_ADMIN_NAME:-db_admin}"
DB_ADMIN_PASS="${DB_ADMIN_PASS:-$(head -n50 '/dev/urandom' | tr -dc 'a-zA-Z0-9' | tr -d '[:space:]\042\047\134' | fold -w 32 | head -n 1)}"
DB_USER_NAME="${DB_USER_NAME:-$USER}"
DB_USER_PASS="${DB_USER_PASS:-$(head -n50 '/dev/urandom' | tr -dc 'a-zA-Z0-9' | tr -d '[:space:]\042\047\134' | fold -w 32 | head -n 1)}"
```

## APP USER ENV

```shell
APP_RUN_AS=""
APP_TEMP_PASS="${APP_TEMP_PASS:-$(head -n50 '/dev/urandom' | tr -dc 'a-zA-Z0-9' | tr -d '[:space:]\042\047\134' | fold -w 32 | head -n 1)}"
APP_USER_NAME="${APP_USER_NAME:-$USER}"
APP_USER_PASS="${APP_USER_PASS:-$(head -n50 '/dev/urandom' | tr -dc 'a-zA-Z0-9' | tr -d '[:space:]\042\047\134' | fold -w 32 | head -n 1)}"
APP_ADMIN_USER="${APP_ADMIN_USER:-${USER}_admin}"
APP_ADMIN_PASS="${APP_ADMIN_PASS:-$(head -n50 '/dev/urandom' | tr -dc 'a-zA-Z0-9' | tr -d '[:space:]\042\047\134' | fold -w 32 | head -n 1)}"
```

## TOKEN VARS

```shell
APP_JWT_TOKEN="${APP_JWT_TOKEN:-$(openssl rand -hex 32)}"
APP_API_TOKEN="${APP_API_TOKEN:-$(openssl rand -hex 16)}"
APP_SECRET_TOKEN_16="${APP_SECRET_TOKEN_16:-$(openssl rand -hex 16)}"
APP_SECRET_TOKEN_32="${APP_SECRET_TOKEN_32:-$(openssl rand -hex 32)}"
APP_SECRET_TOKEN_64="${APP_SECRET_TOKEN_64:-$(openssl rand -hex 64)}"
APP_SECRET_TOKEN_DEFAULT="${APP_SECRET_TOKEN_DEFAULT:-$(openssl rand -hex 16)}"
SECURE_SECRET="${SECURE_SECRET:-$(openssl rand --hex 32)}"
K256_PRIVATE_KEY="${K256_PRIVATE_KEY:-$(openssl ecparam --name secp256k1 --genkey --noout --outform DER | tail --bytes=+8 | head --bytes=32 | xxd --plain --cols 32)}"
```

## KEY VARS

```shell
RPC_SECRET="${RPC_SECRET:-$(openssl rand -hex 32)}"
ENCRYPTION_KEY="${ENCRYPTION_KEY:-$(openssl rand -hex 32)}"
```

## E-Mail ENV

```shell
EMAIL_SERVER_DOMAIN=""
EMAIL_SERVER_HOST=""
EMAIL_SERVER_PORT=""
EMAIL_SERVER_ENCRYPTION=""
EMAIL_SERVER_TIMEOUT="10"
EMAIL_SERVER_FROM_NAME=""
EMAIL_SERVER_FROM_EMAIL=""
EMAIL_SERVER_LOGIN_NAME=""
EMAIL_SERVER_LOGIN_PASS=""
```

## DNS RFC2136 credentials

```shell
RFC2136_NAMESERVER=
RFC2136_TSIG_ALGORITHM=
RFC2136_TSIG_KEY=certbot.
RFC2136_TSIG_SECRET=
RFC2136_PROPAGATION_TIMEOUT=90
```


## Create random password

```shell
head -n50 '/dev/urandom' | tr -dc 'a-zA-Z0-9' | tr -d '[:space:]\042\047\134' | fold -w 32 | head -n 1
```

## Create hash password

```shell
openssl passwd -1 -salt $(openssl rand -base64 6) $(tr -dc A-Za-z0-9 <"/dev/urandom" | head -c 20)
```

## Create secret

```shell
openssl rand -hex 32
```

## Create JWT secret

```shell
openssl rand -base64 32
```

## Author  

🤖 casjay: [Github](https://github.com/casjay) 🤖  
