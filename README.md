# TSM-GNS-APP-Perf

# Prepare Cluster #
prepare cluster  
Install Istio  
install prometheus step by step https://devopscube.com/setup-prometheus-monitoring-on-kubernetes/  
git clone https://github.com/istio/tools.git  

kubernetes create ns fortioclient    
kubernetes create ns fortioserver  
kubernetes -n fortioclient apply -f twopods/fortio_test_client.yaml  
kubernetes -n fortioserver apply -f twopods/fortio_test_server.yaml  

# Prepare Python Environment #
cd tools/perf/benchmark  
cd runner  
pipenv shell  
pipenv install  
cd ..  

# Run performance tests #
export NAMESPACE=fortioclient  
export DNS_DOMAIN=local  
python3 ./runner/runner.py --conn 1,2,4,8,16,32,64 --qps 1000 --duration 240 --baseline --telemetry_mode telemetryv2  
python3 ./runner/runner.py --conn 16 --qps 100,200,400,800,1000 --duration 240 --baseline --telemetry_mode telemetryv2  

# Gather Result Metrics #
kubernetes -n monitoring port-forward prometheus-deployment-7bb6c5d7fd-mb9dm 9090:9090 &  
kubernetes get svc -n fortioclient fortioclient // get external_ip  
export FORTIO_CLIENT_URL=http://external_ip:8080  
export PROMETHEUS_URL=http://localhost:9090  
python3 ./runner/fortio.py $FORTIO_CLIENT_URL --prometheus=$PROMETHEUS_URL --csv StartTime,ActualDuration,Labels,NumThreads,ActualQPS,p50,p90,p99,cpu_mili_avg_telemetry_mixer,cpu_mili_max_telemetry_mixer,mem_MB_max_telemetry_mixer,cpu_mili_avg_fortioserver_deployment_proxy,cpu_mili_max_fortioserver_deployment_proxy,mem_MB_max_fortioserver_deployment_proxy,cpu_mili_avg_ingressgateway_proxy,cpu_mili_max_ingressgateway_proxy,mem_MB_max_ingressgateway_proxy  

# Example of Echo/Http/Grpc Ping #
kubernetes -n fortioclient exec fortioclient-POD -- fortio curl http://fortioserver.gns-sc.local:8080  
kubernetes -n fortioclient exec fortioclient-POD -- fortio load -a -grpc -ping -grpc-ping-delay 0.25s -payload "01234567890" -c 2 -s 4 -labels grpc http://fortioserver.gns-sc.local:8076  
kubernetes -n fortioclient exec fortioclient-POD -- fortio load  -jitter=False -c 2 -qps 1000 -t 93s -a -r 0.001   -httpbufferkb=128 -labels bb http://fortioserver.gns-sc.local:8077/echo?size=1024  

# Visualize Results #
cd graph_plotter/  
python3 ./graph_plotter.py --graph_type=latency-p999 --x_axis=conn --telemetry_modes=telemetryv2_both,telemetryv2_baseline --query_list=1,2,4,8,16,32,64 --query_str=ActualQPS==1000 --csv_filepath=250PODS_SC.csv --graph_title=p999.png  

python3 ./graph_plotter.py --graph_type=ingressgateway-cpu --x_axis=qps --telemetry_modes=telemetryv2_both,telemetryv2_baseline --query_list=10,100,200,400,800,1000 --query_str=NumThreads==16 --csv_filepath=250PODS_SC.csv --graph_title=cpu.png  



