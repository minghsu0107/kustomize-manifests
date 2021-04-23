# Blackbox Exporter
Example Prometheus scraping configurations:
```yaml
scrape_configs:
  ...
  - job_name: 'healthcheck-url'
    metrics_path: /probe
    params:
      module: [http_2xx]  # Look for a HTTP 200 response.
    static_configs:
      - targets:
        - https://google.com
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter.default:9115

  - job_name: 'stripe-healthcheck'
    metrics_path: /probe
    params:
      module: [http_2xx_for_stripe_api]
    static_configs:
      - targets:
        - https://api.stripe.com/v1/charges
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter.default:9115
  ...
```