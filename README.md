# 🧑🏼‍🍳🍲 OpenTelemetry Collector Cookbook

This is a curated collection of OpenTelemetry Collector recipes for use with Netdata.

## 📔 Recipes

Each directory in this repository is a recipe and has an appropriate readme file with instructions.

## 🍴 Tools Used

- The [Netdata OpenTelemetry Plugin (`otel.plugin`)](https://github.com/netdata/netdata/tree/master/src/crates/netdata-otel/otel-plugin) provides an ingestion endpoint for Netdata Agent.
- The [OpenTelemetry Collector Contrib](https://github.com/open-telemetry/opentelemetry-collector-releases/blob/main/distributions/otelcol-contrib/README.md) distribution. This is the main distribution by the OpenTelemetry project. Derived distributions by other vendors will likely work just as well.

### 🗄️📈📄⚠️ 🤖🧰 Netdata 

The `otel.plugin` to Netdata Agent facilitates the storage of signals (metrics and logs for now), and make them available for viewing, alerting, AI investigations, etc. It ships with all of the Netdata Agent packages for Linux based platforms and accepts OTLP over RPC on port 4317 by default.

### 🪟 Windows

The Netdata Agent for Windows does not include the `otel.plugin`, so on Windows the Collector sends its data to a Netdata Agent on a Linux host.

The Collector's MSI installer (from the [releases page](https://github.com/open-telemetry/opentelemetry-collector-releases/releases)) installs it as a Windows service, `otelcol-contrib`, that runs as `LocalSystem`. The installer also puts a stock `config.yaml` in its installation folder, `C:\Program Files\OpenTelemetry Collector`. Do not edit that file: it is meant to stay the stock configuration, so [every upgrade replaces it](https://github.com/open-telemetry/opentelemetry-collector/issues/10043) and an uninstall removes it. Keep your configuration in `%ProgramData%\OpenTelemetry Collector` instead, and point the service at it with the `COLLECTOR_SVC_ARGS` installer property. From an elevated PowerShell, in the folder with your configuration (here `otelcol.yaml`) and the downloaded MSI (adjust the file name to its version):

```powershell
New-Item -ItemType Directory -Force "$env:ProgramData\OpenTelemetry Collector" | Out-Null
Copy-Item .\otelcol.yaml "$env:ProgramData\OpenTelemetry Collector\config.yaml"
Start-Process msiexec.exe -Wait -ArgumentList '/i otelcol-contrib_0.165.0_windows_x64.msi /qn COLLECTOR_SVC_ARGS="--config ""C:\ProgramData\OpenTelemetry Collector\config.yaml"""'
```

The service starts right away. After later changes to `C:\ProgramData\OpenTelemetry Collector\config.yaml`, run `Restart-Service otelcol-contrib`.

> [!WARNING]
> Run the same command, with the newer MSI, to upgrade. An upgrade without `COLLECTOR_SVC_ARGS` switches the service back to the stock configuration: it then sends nothing to Netdata, and it listens for OTLP and Jaeger traffic on all network interfaces. Running the same MSI again does not change the service; to recover, uninstall it (`msiexec /x <msi> /qn`) and install it again with the command above. The installer leaves your configuration and anything else the Collector keeps in `%ProgramData%` alone.

The service writes its own log messages to the Application event log, under the `otelcol-contrib` source.

### 🧂 debug exporter

Most examples include pipelines that export to the `debug` exporter for easier debugging while preparing the recipe. It can be removed from the configuration in the `exporters` section and from the pipelines before serving, to taste. On Windows, keep it to a Collector running interactively in a console: the service's log is the Application event log, which the debug output would flood.

## 📢 Credits

This cookbook was modeled after [jpkrohling's cookbook](https://github.com/jpkrohling/otelcol-cookbook).
