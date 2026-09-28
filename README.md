# freegeoip

[![Docker Pulls](https://img.shields.io/docker/pulls/techblog/freegeoip)](https://hub.docker.com/r/techblog/freegeoip)
[![License](https://img.shields.io/badge/license-Apache%202.0-blue)](License)

A Docker image for [freegeoip](https://github.com/fiorix/freegeoip), a self-hosted HTTP API for looking up the geolocation of IP addresses. freegeoip answers from a local MaxMind GeoLite2 City database, returning the country, region, city, time zone, latitude and longitude for an IP address or host name.

This repository only packages the upstream freegeoip release (version 3.4.1) with a GeoLite2 database; it does not change freegeoip itself.

## Contents

| File | Description |
|------|-------------|
| [`Dockerfile`](Dockerfile) | Alpine image that downloads the freegeoip 3.4.1 `linux-amd64` release and adds the GeoLite2 database. |
| [`docker-compose.yml`](docker-compose.yml) | Example Compose service. |
| [`freegeoip.service`](freegeoip.service) | systemd unit for running freegeoip directly on a host from `/opt/freegeoip`. |
| [`VERSION`](VERSION) | Image version used for the Docker Hub tag. |

## Docker image

| Tag | Status |
|-----|--------|
| `techblog/freegeoip:latest`, `techblog/freegeoip:1.0.0` | Published on Docker Hub (January 2022) for `linux/amd64`, `linux/arm64` and `linux/arm/v7`. |
| `1.1.0`, `1.2.0`, `1.3.0` (current `VERSION`) | Never published as images (a `1.1.0` GitHub release exists, but its source no longer contains the database the Dockerfile copies). |

The published image contains the GeoLite2 database from January 2022, so its location data is out of date.

> **Note:** the Dockerfile always downloads the `linux-amd64` freegeoip binary. The published arm64 and arm/v7 images contain that x86-64 binary, so they do not run on ARM hosts without x86 emulation.

## Quick start

```yaml
# compose.yaml
services:
  freegeoip:
    image: techblog/freegeoip
    container_name: freegeoip
    ports:
      - "8080:8080"
      - "8888:8888"
    restart: always
```

```bash
docker compose up -d
```

The repository's own [`docker-compose.yml`](docker-compose.yml) uses the legacy Compose v1 format (no `services:` key), which current `docker compose` rejects. Use the snippet above, or run the image directly:

```bash
docker run -d --name freegeoip -p 8080:8080 -p 8888:8888 techblog/freegeoip
```

## Usage

The public API listens on port **8080** and the internal metrics server on port **8888**.

```bash
$ curl -s http://localhost:8080/json/8.8.8.8 | jq .
{
  "ip": "8.8.8.8",
  "country_code": "US",
  "country_name": "United States",
  "region_code": "",
  "region_name": "",
  "city": "",
  "zip_code": "",
  "time_zone": "America/Chicago",
  "latitude": 37.751,
  "longitude": -97.822,
  "metro_code": 0
}
```

freegeoip also serves `/csv/<ip>` and `/xml/<ip>`, and a lookup without an address (for example `/json/`) looks up the address the request came from. Behind Docker's NAT or a reverse proxy that is the gateway or proxy address, not the client: the image does not pass freegeoip's `-use-x-forwarded-for` flag. See the [freegeoip documentation](https://github.com/fiorix/freegeoip) for all options. The static files from the release's `public/` folder are served at `/`.

### Metrics

Prometheus metrics are available on the internal server:

```bash
$ curl -s http://localhost:8888/metrics
freegeoip_client_connections{proto="http"} 0
freegeoip_client_country_code_total{country_code="unknown"} 7
freegeoip_client_ipproto_version_total{ip="4"} 7
freegeoip_db_events_total{event="loaded"} 1
go_gc_duration_seconds{quantile="0"} 5.9754e-05
...
```

## Building the image

The Dockerfile expects the database at `data/GeoLite2-City.mmdb.gz`, which is **not** included in this repository. Download the GeoLite2 City database from [MaxMind](https://dev.maxmind.com/geoip/geolite2-free-geolocation-data) (a free account is required), extract `GeoLite2-City.mmdb` from the downloaded archive and gzip it to `data/GeoLite2-City.mmdb.gz`, then build:

```bash
docker build -t freegeoip:local .
```

## Running without Docker

`freegeoip.service` runs `/opt/freegeoip/freegeoip -public public -http :8080 -internal-server :8888 -db data/GeoLite2-City.mmdb.gz` with `/opt/freegeoip` as the working directory. Extract the freegeoip `linux-amd64` release there with `tar xzf freegeoip-3.4.1-linux-amd64.tar.gz --strip-components 1 -C /opt/freegeoip` (upstream publishes amd64 binaries only), add the database under `data/`, copy the unit to `/etc/systemd/system/`, then run `systemctl enable --now freegeoip`.

## CI

| Workflow | Trigger | Publishes |
|----------|---------|-----------|
| [`docker-publish.yml`](.github/workflows/docker-publish.yml) | GitHub release published | `techblog/freegeoip:latest` and `:<VERSION>` for amd64, arm64 and arm/v7 |
| [`publish-ghcr.yml`](.github/workflows/publish-ghcr.yml) | Manual | `ghcr.io/t0mer/freegeoip:latest` and `:<tag input>` (no public image yet) |

## Credits

* [freegeoip](https://github.com/fiorix/freegeoip) by Alexandre Fiori and contributors.
* The Dockerfile is based on the EasyPi Software Foundation's [freegeoip image](https://github.com/vimagick/dockerfiles/tree/master/freegeoip).
* IP geolocation data from [MaxMind GeoLite2](https://dev.maxmind.com/geoip/geolite2-free-geolocation-data).

## License

The packaging in this repository is licensed under the [Apache License 2.0](License). freegeoip and the GeoLite2 database are covered by their own licenses.
