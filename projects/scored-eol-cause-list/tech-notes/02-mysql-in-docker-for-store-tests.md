# Tech note — MySQL in Docker for store tests

**Type:** Tech note
**Project:** CASE — Scored EOL cause list
**Date:** 2026-11-11
**Decided by:** Jonathan, with Vector Elon

Issue 02 stores the spike in MySQL. Flyway needs a database. The lock check and the transaction check need a real database. This note records how CASE gets that database. It does not choose a k3s cluster or namespace.

## Decision

1. CASE uses MySQL `8.0.21`. That version matches A-MES k3s and CI. Compose and Testcontainers use the same image tag.
2. Do not install MySQL on the Ubuntu host for this work.
3. Use Testcontainers inside the Spring Boot tests. The test starts MySQL in Docker, runs the check, then stops MySQL.
4. Use an optional Docker Compose MySQL for hand diagnose and a manual Flyway run. That container is separate from Testcontainers. File: `case/docker/mysql/docker-compose.yml`.
5. Point Flyway at the k3s database only when CASE deploys. Do not use the k3s database as the daily test database.
6. Park the k3s cluster and namespace choice until deploy. Flyway for local and test work does not need that choice.

## Why

A host MySQL install stays forever and drifts from CI. A shared k3s database collides across developers and leaves junk after a failed lock test. Testcontainers gives each test run a private MySQL. Compose gives Jonathan one open MySQL when he wants to poke rows by hand.

## What Docker Compose is

Docker Compose is a small file that starts one or more Docker containers together. CASE would use one service: MySQL. One command starts it. One command stops it.

## Install Docker Engine and Compose (Ubuntu)

The steps below install Docker Engine and Compose on Ubuntu.

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo usermod -aG docker "$USER"
```

Log out and log in after the group change. Then check:

```bash
docker --version
docker compose version
docker info
```

`docker info` must show a **Server** section. If only the Client section prints, the daemon is down or this shell cannot reach `/var/run/docker.sock`.

## IntelliJ and Testcontainers

`SpikeServiceIT` starts MySQL through Testcontainers. The IDE process must reach the same Docker daemon as your terminal.

If IntelliJ fails with `DOCKER_HOST unix:///var/run/docker.sock is not listening`, `Permission denied`, or `Could not find a valid Docker environment`, while a terminal `docker info` works:

1. Confirm Docker from a terminal: `docker info` shows Server.
2. Fully quit IntelliJ.
3. Start IntelliJ from that same terminal, for example `idea` or your Toolbox CLI.
4. Run `SpikeServiceIT` again.

Do not rely on an old desktop launcher session if that process started before your user joined the `docker` group. The shell that can talk to Docker must be the parent of the IDE process.

Compose MySQL is optional for hand diagnose. Testcontainers does not need that Compose stack.

## Optional Compose MySQL for hand diagnose

File in the CASE app repo: `docker/mysql/docker-compose.yml`.

Start:

```bash
cd docker/mysql
docker compose up -d
```

JDBC URL for hand work: `jdbc:mysql://localhost:3306/case`  
User `case-user`, password `case`. Root password `case-root`. The app uses `case-user`. Do not point the Spring Boot datasource at MySQL root.

Open a SQL shell:

```bash
docker exec -it case-mysql mysql -ucase-user -pcase case
```

Stop and keep data:

```bash
docker compose stop
```

Remove the container and keep the volume:

```bash
docker compose down
```

Wipe data too:

```bash
docker compose down -v
```

## Testcontainers vs Compose

| Path | Who starts MySQL | When to use |
|---|---|---|
| Testcontainers | The Spring Boot test | Automated store checks, transaction check, lock check |
| Docker Compose | Jonathan by hand | Local diagnose, manual Flyway against a standing MySQL |
| k3s database | Cluster deploy | Deploy migrate only |

These are not two Testcontainers stacks. Testcontainers is for tests. Compose is optional for hand work.

## Still open

- Passwords in the Compose file stay local-only. Do not ship them to plant.
- k3s cluster and namespace stay parked until deploy.

## Links

- Contract keys and locks: [02-contract-store-the-spike.md](../design/02-contract-store-the-spike.md)
- API store design: [02-api-store-the-spike.md](../design/02-api-store-the-spike.md)
- Issue: [02-the-operator-submits-the-product-and-the-eol-stations.md](../issues/02-the-operator-submits-the-product-and-the-eol-stations.md)
