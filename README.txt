1. Install and deploy Nginx
# Creating a dedicated network is necessary as by default rootless containers
# are isolated. Creating a network give us the ability to not map the container
# port to a host port and simplify the connection from the mariadb client to the
# mariadb server.
podman network create db
podman run -it --rm -d -p 8090:80 --network db --name web -v ./client:/usr/share/nginx/html docker.io/nginx:alpine

2. Install and deploy MariaDB
# We can use a volume to pass a SQL script at /docker-entrypoint-initdb.d/;
# only the GRANT statement works (DB and user creation is not effective this
# way). Passing environmental variables to podman in order to create a DB and a
# user is effective. Anyhow, such script is useless as the default privileges of
# a user created by invoking the environmental variabale is as a super user.
podman run -it -d --rm --network db --name db --env MARIADB_USER=manning --env MARIADB_PASSWORD=WasmEdge --env MARIADB_DATABASE=umet --env MYSQL_ROOT_PASSWORD=whalehello docker.io/mariadb:latest
# podman run -it --rm --network db --name client-db docker.io/mariadb:latest mariadb -hdb -p
# MariaDB [(none)]> \s
# MariaDB [(none)]> GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, INDEX, DROP, ALTER, CREATE TEMPORARY TABLES, LOCK TABLES ON umet.* TO manning ;
# MariaDB [(none)]> \q
# podman run -it --rm --network db --name client-db docker.io/mariadb:latest mariadb -hdb -umanning -p
# MariaDB [(none)]> \s
# MariaDB [(none)]> \q

3. Start the two microservices
cd sales_tax_rate_lookup
podman build --annotation module.wasm.image/variant=compat -t docker.io/baugereau/sales_tax_rate_lookup.wasm -f Containerfile
cd ..
podman run --rm --annotation module.wasm.image/variant=compat baugereau/sales_tax_rate_lookup.wasm:latest
cd order_management
podman build --annotation module.wasm.image/variant=compat -t docker.io/baugereau/order_management.wasm -f Containerfile
cd ..
# open a new terminal
podman network inspect db # get the gateway IP of the created network to pass as environment variable to the order container
podman run --rm -d --network db --name order --env  "SALES_TAX_RATE_SERVICE=http://10.89.0.1:8001/find_rate" --env "DATABASE_URL=mysql://manning:WasmEdge@10.89.0.1:3306/umet" docker.io/baugereau/order_management.wasm:latest

4. Open your browser at http://localhost/ to test the microservice through its web UI.

5. Use the mysql command to create a new username, password, and empty database. Restart the microservices to test and verify that they now save data into the new database.

