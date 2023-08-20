ziti edge login "${ZITI_CTRL_EDGE_ADVERTISED_ADDRESS}:${ZITI_CTRL_EDGE_ADVERTISED_PORT}" -u $ZITI_USER -p $ZITI_PWD 

ziti edge delete identity prometheus.scrape.test.user

rm *target*json *target*jwt
ziti edge delete identity target1.prometheus.scrape.target target2.prometheus.scrape.target target3.prometheus.scrape.target

ziti edge create identity device target1.prometheus.scrape.target -o target1.prometheus.scrape.target.jwt -a prometheus.scrape.targets
ziti edge create identity device target2.prometheus.scrape.target -o target2.prometheus.scrape.target.jwt -a prometheus.scrape.targets
ziti edge create identity device target3.prometheus.scrape.target -o target3.prometheus.scrape.target.jwt -a prometheus.scrape.targets
ziti edge create identity device prometheus.scrape.test.user -o prometheus.scrape.test.user.jwt -a prometheus.scrapers

./ziti-edge-tunnel-0.21.5 enroll -j target1.prometheus.scrape.target.jwt -i target1.prometheus.scrape.target.json
./ziti-edge-tunnel-0.21.5 enroll -j target2.prometheus.scrape.target.jwt -i target2.prometheus.scrape.target.json
./ziti-edge-tunnel-0.21.5 enroll -j target3.prometheus.scrape.target.jwt -i target3.prometheus.scrape.target.json

chmod 777 *.prometheus.scrape.target.json

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







