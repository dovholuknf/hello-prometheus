# Prometheus and OpenZiti

This repo was inspired by the blog post: https://codeburst.io/prometheus-by-example-4804ab86e741 which links to the
source repo this one was forked from: https://github.com/larkintuckerllc/hello-prometheus. It is a small repo that
starts up three docker networks, each with an apache server running and a Prometheus
apache exporter. It then uses an OpenZiti Network to provide reach into each of the docker networks in order to
scrape the exporters in each network. This will emulate a centralized Prometheus server scraping three remote sites.
This is particularly useful as the exporters will not need to be exposed to the open internet. No inbound firewall
rules needed which is generally necessary to enable Prometheus to scrape the exporters.

Here's the list of things that need to be accomplished. Start by cloning this repo and cd'ing to the checkout:
1. Install a OpenZiti Network which will be addressable from within Docker and from wherever you want/need. It'll be
   easiest to deploy this on the open internet if possible just to make sure this isn't an issue.

1. set four shell variables:
   * set `ZITI_CTRL_EDGE_ADVERTISED_ADDRESS`
   * set `ZITI_CTRL_EDGE_ADVERTISED_PORT`
   * set `$ZITI_USER`
   * set `$ZITI_PWD`
   
1. Use the `ziti` CLI to login:

       ziti edge login "${ZITI_CTRL_EDGE_ADVERTISED_ADDRESS}:${ZITI_CTRL_EDGE_ADVERTISED_PORT}" -u $ZITI_USER -p $ZITI_PWD 
1. Delete any existing OpenZiti configuration:

       ziti edge delete identity prometheus.scrape.test.user
       rm *target*json *target*jwt
       ziti edge delete identity target1.prometheus.scrape.target target2.prometheus.scrape.target target3.prometheus.scrape.target
       
1. Configure the overlay by creating three identities for each exporter and one identity to use to test with

       ziti edge create identity device target1.prometheus.scrape.target -o target1.prometheus.scrape.target.jwt -a prometheus.scrape.targets
       ziti edge create identity device target2.prometheus.scrape.target -o target2.prometheus.scrape.target.jwt -a prometheus.scrape.targets
       ziti edge create identity device target3.prometheus.scrape.target -o target3.prometheus.scrape.target.jwt -a prometheus.scrape.targets
       ziti edge create identity device prometheus.scrape.test.user -o prometheus.scrape.test.user.jwt -a prometheus.scrapers

1. Enroll the Prometheus exporter identities
       
       ./ziti-edge-tunnel enroll -j target1.prometheus.scrape.target.jwt -i target1.prometheus.scrape.target.json
       ./ziti-edge-tunnel enroll -j target2.prometheus.scrape.target.jwt -i target2.prometheus.scrape.target.json
       ./ziti-edge-tunnel enroll -j target3.prometheus.scrape.target.jwt -i target3.prometheus.scrape.target.json

1. For ease, chmod the identity files to 777 for using within the Docker containers

       chmod 777 *.prometheus.scrape.target.json

1. Create the needed services, configs and service policies

       ziti edge create config "prometheus.intercept.v1" intercept.v1 \
       '{"protocols":["tcp"],"addresses":["*.prometheus.scrape.target"], "portRanges":[{"low":9117, "high":9117}], "dialOptions":{"identity":"$dst_hostname"}}'
       
       ziti edge create config "prometheus.host.v1" host.v1 \
       '{"protocol":"tcp", "address":"apache-exporter","port":'9117', "listenOptions": {"bindUsingEdgeIdentity":true}}'
       
       ziti edge create service prometheus.svc \
       --configs "prometheus.intercept.v1,prometheus.host.v1"
       
       ziti edge create service-policy prometheus.svc.bind Bind \
       --service-roles "@prometheus.svc" \
       --identity-roles "#prometheus.scrape.targets"
       
       ziti edge create service-policy prometheus.svc.dial Dial \
       --service-roles "@prometheus.svc" \
       --identity-roles "#prometheus.scrapers"

1. Clean and then launch the docker compose test environment

       docker compose down -v; docker compose up

1. Let the docker compose environment run for a while then access the results at 

   http://localhost:9090/graph?g0.expr=apache_accesses_total&g0.tab=0&g0.stacked=0&g0.show_exemplars=0&g0.range_input=1h


