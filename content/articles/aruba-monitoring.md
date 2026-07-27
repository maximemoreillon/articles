---
date: '2026-06-15'
title: "Monitoring an Aruba WiFi access point with Prometheus and Grafana"
tags: ['Monitoring', 'Grafana', 'Prometheus']
---

I currently use an Aruba IAP-315 (ArubaOS v8) as WiFi access point. Although it provides insight such as the number connected clients on its own web interface, I wanted to visualize this data in Grafana.

This can be achieved thanks to Aruba access points' support of the SNMP protocol, which can be exported as Prometheus metrics using the [Prometheus SNMP Exporter](https://github.com/prometheus/snmp_exporter/tree/main).

This article focuses simply on client count but more data can be extracted in similar fashion.

## Aruba IAP configuration

SNMP can be enabled by creating a SNMPV3 user through the web UI. The settings are located under `Configuration > System > Monitoring > Users for SNMPV3` (with advanced options shown), where the `+` button opens the user creation dialog. Configuration is as follows:

- name: a username of your choice
- Authentication protocol: SHA
- Password: password of your choice
- Privacy Protocol: AES
- Passsword: password of your choice

## `snmpwalk`

The `snmpwalk` command can be used to figure out what data the IAP is willing to tell about itself:

```bash
snmpwalk -v3 -l authPriv \
  -u <YOUR USERNAME> -a SHA -A <YOUR AUTH PASSWORD> \
  -x AES -X <YOUR PRIVACY PASSWORD> \
  <IP OF THE ARUBA CONTROLLER> 1.3.6.1.4.1.14823.2.3.3
```

The output contains a notable lines such as

```
SNMPv2-SMI::enterprises.14823.2.3.3.1.2.2.1.21.36.242.127.195.244.230.1 = INTEGER: 24
```

where 24 happens to be the client count of my SSID.

This information will become useful when configuring the Prometheus SNMP Exporter.

## Prometheus SNMP Exporter

My infrastructure runs on Kubernetes so I deployed the SNMP Exporter as a `Deployment` object with the following manifest:

```yml
# Deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: snmp-exporter
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: snmp-exporter
  template:
    metadata:
      labels:
        app: snmp-exporter
    spec:
      containers:
        - name: snmp-exporter
          image: prom/snmp-exporter:v0.26.0
          args:
            - --config.file=/etc/snmp_exporter/snmp.yml
          volumeMounts:
            - name: config
              mountPath: /etc/snmp_exporter
      volumes:
        - name: config
          secret:
            secretName: snmp-exporter-config
```

The configuration is defined in a `Secret` object as SNMP Exporter does not substitute environment variables in `snmp.yml`.
The settings under `modules.walk` and `modules.metrics` are derived from the `snmpwalk` command hereabove.

```yml
# config-secret.yml
apiVersion: v1
kind: Secret
metadata:
  name: snmp-exporter-config
  namespace: monitoring
type: Opaque
stringData:
  snmp.yml: |
    auths:
      aruba_v3:
        security_level: authPriv
        username: <YOUR USERNAME>
        password: <YOURAUTH PASSWORD>
        auth_protocol: SHA
        priv_protocol: AES
        priv_password: <YOUR PRIVACY PASSWORD>
        version: 3
    modules:
      if_mib:
        walk:
          - 1.3.6.1.2.1.2        # interfaces
          - 1.3.6.1.2.1.31.1.1   # ifXTable

          - 1.3.6.1.4.1.14823.2.3.3.1.1.7.1.4   # clients per SSID

        metrics:
        - name: aruba_ssid_client_count
          oid: 1.3.6.1.4.1.14823.2.3.3.1.1.7.1.4
          type: gauge
          help: Number of clients associated to this SSID
          indexes:
            - labelname: ssid_index
              type: gauge
```

A `Service` object handles networking to the application:

```yaml
# svc.yml
apiVersion: v1
kind: Service
metadata:
  name: snmp-exporter
  namespace: monitoring
spec:
  selector:
    app: snmp-exporter
  ports:
    - port: 9116
  type: ClusterIP
```

With SNMP exporter deployed, the exposed metrics can be checked using a port-forward:

```bash
kubectl port-forward -n monitoring svc/snmp-exporter 9116
```

After which the metrics endpoint `/snmp` gets exposed on localhost:

```bash
curl "http://localhost:9116/snmp?target=YOUR_ARUBA_CONTROLLER_IP&auth=aruba_v3&module=if_mib"
```

which should return various metrics, including that of the client count:

```
# HELP aruba_ssid_client_count Number of clients associated to this SSID
# TYPE aruba_ssid_client_count gauge
aruba_ssid_client_count{ssid_index="0"} 24
```

## Prometheus

With the metrics properly exported, Prometheus can be configured for scraping. This can be achieved with the following configuration:

```yml
- job_name: snmp-exporter
  metrics_path: /snmp
  scheme: http
  params:
    module: [if_mib]
    auth: [aruba_v3]
  static_configs:
    - targets:
        - YOUR ARUBA CONTROLLER IP
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: snmp-exporter.monitoring:9116
```

## Conclusion

By leveraging Aruba WiFI accesss points' support for SNMP and exporting metrics with the SNMP Exporter, information about SSIDs can be scraped by Prometheus.
This data can then be visualized in Grafana for example.
