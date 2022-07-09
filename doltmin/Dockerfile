# syntax = docker/dockerfile:1.4
FROM golang:alpine as builder

RUN apk update
RUN apk add git

RUN git clone https://github.com/dolthub/dolt.git /var/dolt
RUN cd /var/dolt/go && echo "replace github.com/dolthub/go-mysql-server => /var/go-mysql-server" >> go.mod

RUN git clone https://github.com/dolthub/go-mysql-server.git /var/go-mysql-server
RUN cd /var/go-mysql-server && git checkout fulghum/alterpk-panic

RUN cd /var/dolt/go && go install ./cmd/dolt


# ----------------------------------------------------------------------------

FROM alpine:latest

COPY --from=builder /go/bin/dolt /bin/dolt
ENV PATH="/bin:$PATH"

# Set default variables
ENV DOLT_HOST="0.0.0.0"
ENV DOLT_PORT=3306
ENV DOLT_LOGLEVEL="debug"
ENV DOLT_DB="doltdb"
ENV DOLT_ROOT_USER="root"
ENV DOLT_ROOT_PASSWORD="root"
ENV VERBOSE_STARTUP=1

COPY <<-EOF /startup.sh
#!/bin/ash
set -eo pipefail

# fail a step and print the message
_fail() {
  local message="\$1"
  echo "\$message"
  exit 1;
}

# check a single environment variable value
_check_var() {
  local key="\$1"
  local val="\$2"

  if [ "\$val" == "" ]; then
    _fail "Must set \$key"
  fi
}

_create_dir() {
  local path="\$1"
  mkdir -p "\$path"
}

_say() {
  if [ "\$VERBOSE_STARTUP" == "1" ]; then
    echo "\$1"
  fi
}

# create required directories
create_directories() {
  _say "Creating Directories..."
  _create_dir "/\$DOLT_DB"
}



# check that expected environment variables are defined
check_env_vars() {
  _say "Checking Environment Variables..."
  _check_var "DOLT_HOST" "\$DOLT_HOST"
  _check_var "DOLT_PORT" "\$DOLT_PORT"
  _check_var "DOLT_ROOT_USER" "\$DOLT_ROOT_USER"
#  _check_var "DOLT_ROOT_PASSWORD" "\$DOLT_ROOT_PASSWORD"
  _check_var "DOLT_LOGLEVEL" "\$DOLT_LOGLEVEL"
  _check_var "DOLT_DB" "\$DOLT_DB"
}

configure_dolt_server() {
  local dolt_bin=\$(which dolt)
  if [ ! -x "\$dolt_bin" ]; then
      _fail "dolt not found on PATH"
  fi
}


_main() {
  check_env_vars
  create_directories
  configure_dolt_server

  cd "/\$DOLT_DB"
  exec "\$@"
}

_main "\$@"

EOF

RUN chmod +x /startup.sh
ENTRYPOINT [ "/startup.sh" ]

EXPOSE $DOLT_PORT
CMD [ "/bin/sh", "-c", "dolt sql-server \
       --user=$DOLT_ROOT_USER \
       --password=$DOLT_ROOT_PASSWORD \
       --loglevel=$DOLT_LOGLEVEL \
       --host=$DOLT_HOST \
       --port=$DOLT_PORT" ]
