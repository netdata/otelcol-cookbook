# 🧑🏼‍🍳🍲 OpenTelemetry Collector Cookbook

This is a curated collection of OpenTelemetry Collector recipes for use with Netdata.

## 📔 Recipes

Each directory in this repository is a recipe and has an appropriate readme file with instructions.

## 🍴 Tools Used

- The [Netdata OpenTelemetry Plugin (`otel.plugin`)](https://github.com/netdata/netdata/tree/master/src/crates/netdata-otel/otel-plugin) provides an ingestion endpoint for Netdata Agent.
- The [OpenTelemetry Collector Contrib](https://github.com/open-telemetry/opentelemetry-collector-releases/blob/main/distributions/otelcol-contrib/README.md) distribution. This is the main distribution by the OpenTelemetry project. Derived distributions by other vendors will likely work just as well.

### 🗄️📈📄⚠️ 🤖🧰 Netdata 

The `otel.plugin` to Netdata Agent facilitates the storage of signals (metrics and logs for now), and make them available for viewing, alerting, AI investigations, etc. It ships with all of the Netdata Agent packages for Linux based platforms and accepts OTLP over RPC on port 4317 by default.

### 🧂 debug exporter

Most examples include pipelines that export to the `debug` exporter for easier debugging while preparing the recipe. It can be removed from the configuration in the `exporters` section and from the pipelines before serving, to taste.

## 📢 Credits

This cookbook was modeled after [jpkrohling's cookbook](https://github.com/jpkrohling/otelcol-cookbook).
