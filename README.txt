cd sales_tax_rate_lookup
podman build --annotation module.wasm.image/variant=compat -t docker.io/baugereau/sales_tax_rate_lookup.wasm -f Containerfile
cd ..
cd order_management
podman build --annotation module.wasm.image/variant=compat -t docker.io/baugereau/order_management.wasm -f Containerfile
cd ..
# Creating a dedicated network is necessary as by default rootless containers
# are isolated. Creating a network give us the ability to not map the container
# port to a host port and simplify the connection from the mariadb client to the
# mariadb server.
podman network create db
# We can use a volume to pass a SQL script at /docker-entrypoint-initdb.d/;
# only the GRANT statement works (DB and user creation is not effective this
# way). Passing environmental variables to podman in order to create a DB and a
# user is effective. Anyhow, such script is useless as the default privileges of
# a user created by invoking the environmental variabale is as a super user.
podman run -it -d --rm --network db --name db --env MARIADB_USER=manning --env MARIADB_PASSWORD=WasmEdge --env MARIADB_DATABASE=umet --env MYSQL_ROOT_PASSWORD=whalehello docker.io/mariadb:latest
# podman run -it --rm --network db --name client-db docker.io/mariadb:latest mariadb -hdb -p
# MariaDB [(none)]> \s
# MariaDB [(none)]> \q
# MariaDB [(none)]> GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, INDEX, DROP, ALTER, CREATE TEMPORARY TABLES, LOCK TABLES ON umet.* TO manning ;
# podman run -it --rm --network db --name client-db docker.io/mariadb:latest mariadb -hdb -umanning -p
# MariaDB [(none)]> \s
# MariaDB [(none)]> \q

TODO: 
* find how to manage the ip address of the db for connection from the wasm binary (passed as env variable); manual setting as it would be handled in a better way in the next module with dapr
* change the Dockerfiles to Containerfiles (including the wasm binaries for podman)
