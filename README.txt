1. Install and deploy Nginx
# Creating a dedicated network is necessary as by default rootless containers
# are isolated. Creating a network give us the ability to not map all container
# ports to host ports.
podman network create module4
podman run -it --rm -d -p 8090:80 --network module4 --name web -v ./client:/usr/share/nginx/html docker.io/nginx:alpine

2. Install and deploy MariaDB
# We can use a volume to pass a SQL script at /docker-entrypoint-initdb.d/;
# only the GRANT statement works (DB and user creation is not effective this
# way). Passing environmental variables to podman in order to create a DB and a
# user is effective. Anyhow, such script is useless as the default privileges of
# a user created by invoking the environmental variabale is as a super user.
podman run -it -d --rm --network module4 --name db --env MARIADB_USER=manning --env MARIADB_PASSWORD=WasmEdge --env MARIADB_DATABASE=umet --env MYSQL_ROOT_PASSWORD=whalehello docker.io/mariadb:latest
# podman container inspect db --format "{{(index .NetworkSettings.Networks \"module4\").IPAddress}}"
# podman unshare --rootless-netns podman run -it --rm docker.io/mariadb:latest mariadb -h <IP from previous command> -P 3306 --protocol=TCP -p 
# MariaDB [(none)]> \s
# MariaDB [(none)]> GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, INDEX, DROP, ALTER, CREATE TEMPORARY TABLES, LOCK TABLES ON umet.* TO manning ;
# MariaDB [(none)]> \q
# podman unshare --rootless-netns podman run -it --rm docker.io/mariadb:latest mariadb -h <IP from penultimate command> -P 3306 --protocol=TCP -p -u manning umet
# MariaDB [(none)]> \s
# MariaDB [(none)]> \q

3. Start the two microservices
cd sales_tax_rate_lookup
podman build --annotation module.wasm.image/variant=compat -t docker.io/baugereau/sales_tax_rate_lookup.wasm -f Containerfile
cd ..
podman run --rm -d --network module4 --name sale --annotation module.wasm.image/variant=compat baugereau/sales_tax_rate_lookup.wasm:latest
cd order_management
podman build --annotation module.wasm.image/variant=compat -t docker.io/baugereau/order_management.wasm -f Containerfile
cd ..
# There is a DNS with the created network but passing the container name to the Rust code is not effective. To find the container address:
# Find the container IP as the module4 network DNS is not reachable from the host:
podman container inspect sale --format "{{(index .NetworkSettings.Networks \"module4\").IPAddress}}"
podman container inspect db --format "{{(index .NetworkSettings.Networks \"module4\").IPAddress}}"
podman run --rm -d --network module4 -p 8003:8003 --name order ---env  "SALES_TAX_RATE_SERVICE=http://<IP from the penultimate command>:8001/find_rate" --env "DATABASE_URL=mysql://manning:WasmEdge@<IP from previous command>:3306/umet" docker.io/baugereau/order_management.wasm:latest
# Initiate the orders table in the database:
curl http://localhost:8003/init

4. Open your browser at http://localhost/ to test the microservice through its web UI.
# For convenience I mapped the nginx port to the host on port 8090; so open the browwer at http://localhot:8090
curl http://localhost:8003/orders

5. Use the mysql command to create a new username, password, and empty database. Restart the microservices to test and verify that they now save data into the new database.
# This was performed above when starting the db container.

6. GitHub Actions script.
I have no intend to perform tests through GitHub Actions; I'm skipping this step.
