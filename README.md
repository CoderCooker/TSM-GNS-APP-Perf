# TSM-GNS-APP-Perf

# Prepare Cluster #
prepare cluster
Install Istio
install prometheus step by step https://devopscube.com/setup-prometheus-monitoring-on-kubernetes/
git clone https://github.com/istio/tools.git

export NAMESPACE=fortioclient
k create ns $NAMESPACE
export DNS_DOMAIN=local
cd tools/perf/benchmark
setup_test.sh

# Prepare Python Environment #
cd runner
pipenv shell
pipenv install
cd ..

# Run performance tests #
python3 ./runner/runner.py --conn 1,2,4,8,16,32,64 --qps 1000 --duration 240 --baseline --telemetry_mode telemetryv2
python3 ./runner/runner.py --conn 16 --qps 100,200,400,800,1000 --duration 240 --baseline --telemetry_mode telemetryv2

# Gather Result Metrics #
kubernetes -n monitoring port-forward prometheus-deployment-7bb6c5d7fd-mb9dm 9090:9090 &
kubernetes get svc -n fortioclient fortioclient // get external_ip
export FORTIO_CLIENT_URL=http://external_ip:8080
export PROMETHEUS_URL=http://localhost:9090
python3 ./runner/fortio.py $FORTIO_CLIENT_URL --prometheus=$PROMETHEUS_URL --csv StartTime,ActualDuration,Labels,NumThreads,ActualQPS,p50,p90,p99,cpu_mili_avg_telemetry_mixer,cpu_mili_max_telemetry_mixer,mem_MB_max_telemetry_mixer,cpu_mili_avg_fortioserver_deployment_proxy,cpu_mili_max_fortioserver_deployment_proxy,mem_MB_max_fortioserver_deployment_proxy,cpu_mili_avg_ingressgateway_proxy,cpu_mili_max_ingressgateway_proxy,mem_MB_max_ingressgateway_proxy

# Visualize Results #
cd graph_plotter/
python3 ./graph_plotter.py --graph_type=latency-p999 --x_axis=conn --telemetry_modes=telemetryv2_both,telemetryv2_baseline --query_list=1,2,4,8,16,32,64 --query_str=ActualQPS==1000 --csv_filepath=250PODS_SC.csv --graph_title=p999.png

python3 ./graph_plotter.py --graph_type=ingressgateway-cpu --x_axis=qps --telemetry_modes=telemetryv2_both,telemetryv2_baseline --query_list=10,100,200,400,800,1000 --query_str=NumThreads==16 --csv_filepath=250PODS_SC.csv --graph_title=cpu.png



