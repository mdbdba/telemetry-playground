# Docker Compose Telemetry Stack to play with

This started life as one of the grafana agent examples.  It makes for a good spot to test out different configurations for any of the grafana products.

Changes made from the original example (and if that's what you're really looking for, go clone https://github.com/grafana/agent/tree/main/example/docker-compose ):
   * Updated all of the components (agent, loki, cortex, and tempo) to their latest versions.
   * Both Loki and Cortex are running in a multitenant configuration
     * for Loki: fake, loki1, and loki2; fake should be empty, while both of the other tenants are pulling in the same dataset.  This was done so that we can play with pipelines in one of them and compare back without a bunch of fuss.
     * for Cortex, the first one, autoscrape, was created to show how the autoscrape integration works and what data it collects.  The another, avalanche, shows the metrics coming from that load generator.  There is a fake one defined as well with no data.
   * Grafana agent has two log configs so that each set of logs get send to with the correct tenant id (X-Scope-OrgID).
   * There are two nginx proxies, loki1auth and loki2auth, to demonstrate how basic auth works.
   * In Grafana, the data source definitions for tenents have been added. So, although the two that came with this originally (Cortex and Loki) will work (we've added the "fake" tenent to them), they should not find any data.
   * Tempo and it's data source in grafana have been set up with the backend search enabled, because that's a requirement.

## Overall Setup

This directory contains a Docker Compose 3 environment that can be used to experiment with a telemetry stack with a fair set of features.

By default, the following services are exposed:

1. Cortex for storing metrics (localhost:9009)
2. Grafana for visualizing telemetry (localhost:3000)
3. Loki for storing logs (localhost:3100)
4. Tempo for storing traces (localhost:3200)
5. Avalanche for a sample /metrics endpoint to scrape (localhost:9001)
6. Loki1auth for demonstrating basic auth for the loki1 tenant (localhost:8880)
7. Loki2auth for demonstrating basic auth for the loki2 tenant (localhost:8881)

Run the following to bring up the environment:

```
docker-compose --profile=agent up -d
```

By default, the Docker Compose environment doesn't include a Grafana Agent
container. That would let you test the agent externally. We are enabling the included Grafana Agent by passing`agent` to the profiles list: `docker compose --profile=agent up -d`. When running, the Agent exposes its HTTP endpoint at `localhost:12345`. This address can be changed with the `--server.http.address` flag (e.g.,
`--server.http.address=127.0.0.1:8000`).

The Docker Compose environment heavily relies on profiles to enable optional
features. You can pass multiple profiles by passing the flag multiple times:
`docker compose --profile agent --profile integrations up -d`.

## Running Integrations

The Docker Compose environment includes example services to point integrations
at (e.g., mysql, redis, consul, etc.). You can run all integrations using the
`integrations` profile, or run individual services with a profile name matching
the integration (i.e., enabling the `dnsmasq_exporter` profile to specifically
enable the dnsmasq service).

Enabling specific integration profiles is useful when you only want to test a
single integration, as the `integrations` profile can be resource intensive.

## Visualizing

Grafana is exposed at `http://localhost:3000`, and includes some useful
dashboards:

* The `Agent` dashboard gives a very high-level overview of running agents.

* The `Agent Prometheus Remote Write` dashboard visualizes the current state of
  writing metrics to Cortex.

* The `Agent Tracing Pipeline` dashbaord visualizes the current state of the
  tracing pipeline (if spans are being processed).

* The `Agent Operational` dashboard shows resource consumption of Grafana
  Agent. Not all panels will have data here, as they rely on metrics from other
  sources (i.e., cAdvisor).


## Test inserting into different tenants
These commands insert examples 1&3 directly to loki
```
# build the epoch timestamp
ts=$(printf "%s000000000000" $(date '+%s') |cut -c1-19)
# plug that value into the data for the curl cmd
dr="{\"streams\": [{ \"stream\": { \"foo\": \"bar\" }, \"values\": [ [ \"$ts\", \"example1\" ] ] }]}"
curl --header "X-Scope-OrgID: loki1"  -X POST http://localhost:3100/loki/api/v1/push -H "Content-Type: application/json" --data-raw $dr
dr="{\"streams\": [{ \"stream\": { \"foo\": \"bar\" }, \"values\": [ [ \"$ts\", \"example3\" ] ] }]}"
curl --header "X-Scope-OrgID: loki2"  -X POST http://localhost:3100/loki/api/v1/push -H "Content-Type: application/json" --data-raw $dr
```
## Test inserting through the proxies
These commands insert examples 2&4 by way of the proxies
```
# build the epoch timestamp
ts=$(printf "%s000000000000" $(date '+%s') |cut -c1-19)
# plug that value into the data for the curl cmd
dr="{\"streams\": [{ \"stream\": { \"foo\": \"bar\" }, \"values\": [ [ \"$ts\", \"example2\" ] ] }]}"
curl --header "Authorization: Basic bG9raTE6bG9raTFwQHNzdzByZCE="  --header "X-Scope-OrgID: loki1"  -X POST http://localhost:3100/loki/api/v1/push -H "Content-Type: application/json" --data-raw $dr
dr="{\"streams\": [{ \"stream\": { \"foo\": \"bar\" }, \"values\": [ [ \"$ts\", \"example4\" ] ] }]}"
curl --header "Authorization: Basic bG9raTI6bG9raTJwQHNzdzByZCE=" --header "X-Scope-OrgID: loki2"  -X POST http://localhost:3100/loki/api/v1/push -H "Content-Type: application/json" --data-raw $dr
```
## Query the test records
Using any of the loki data sources:
```
{foo="bar"}
```
If things are set up right, you'll see different results in each of the loki data sources.