# 🍜 Recipe: Windows Event Log Ingest

This recipe ingests events from the [Windows Event Log](https://learn.microsoft.com/en-us/windows/win32/wes/windows-event-log), the central log repository on Windows systems. The Windows Event Log receiver subscribes to event log channels (Application and System by default), and the Collector sends the events to a Netdata Agent.

The receiver delivers each event as a structured body. The `transform` processor maps it to the [OpenTelemetry Logs Data Model mapping for Windows Event Log](https://opentelemetry.io/docs/specs/otel/logs/data-model-appendix/#windows-event-log): the rendered message becomes the log body, the computer becomes `host.name`, and the event ID and all other fields become `winlog.*` attributes (named after [Winlogbeat](https://www.elastic.co/docs/reference/beats/winlogbeat/exported-fields-winlog) where it has a name for them). It also derives `service.name` from the event provider, maps the process, thread and user to [Semantic Convention](https://opentelemetry.io/docs/specs/semconv/) attributes, and aligns the severity with the Logs Data Model.

> [!IMPORTANT]
> The Netdata Agent for Windows does not include the OpenTelemetry Plugin. The Collector runs on the Windows host and sends the events to a Netdata Agent on a Linux host (for example, a Netdata Parent) over the network.

> [!NOTE]
> The Windows Event Log receiver is at *beta* stability and Windows-only. This recipe was tested with OpenTelemetry Collector Contrib v0.165.0 on Windows Server 2025.

## 🧄 Ingredients

- OpenTelemetry Collector Contrib for Windows, see the main [README.md](../README.md) for instructions. The MSI installer from the [releases page](https://github.com/open-telemetry/opentelemetry-collector-releases/releases) installs it as the `otelcol-contrib` Windows service.
- The [`otelcol.yaml`](./otelcol.yaml) file from this directory.
- A Netdata Agent on Linux that the Windows host can reach on port 4317.

## 🥣 Preparation

1. Make the Netdata Agent accept remote OTLP senders. Its OpenTelemetry Plugin listens on `127.0.0.1:4317` only by default. On the Netdata host, run `sudo /etc/netdata/edit-config otel.yaml`, then set:

   ```yaml
   receivers:
     otlp:
       protocols:
         grpc:
           endpoint: "0.0.0.0:4317"
   ```

   Restart the Agent (`sudo systemctl restart netdata`). Without TLS, anyone who can reach this port can send logs to Netdata and read the traffic. Use this only on a trusted network, or restrict the port with a firewall. To protect it, see [Securing the connection](#-securing-the-connection) below.

2. Edit `otelcol.yaml` at the locations marked `TODO`:

   1. The `endpoint` of the `otlp` exporter: the host name or IP address of the Netdata Agent.
   2. Optionally enable the Security channel. See [Choosing channels](#-choosing-channels) below.

   The bookmarks (so restarts resume where they left off) and the on-disk sending queue are stored under `%ProgramData%\OpenTelemetry Collector`.

3. Start the Collector using the provided configuration. With the MSI installation, replace the installed configuration and restart the service from an elevated PowerShell:

   ```powershell
   Copy-Item .\otelcol.yaml "C:\Program Files\OpenTelemetry Collector\config.yaml"
   Restart-Service otelcol-contrib
   ```

   > [!WARNING]
   > The MSI owns that `config.yaml`. Upgrading the Collector with a newer MSI silently replaces it with the stock configuration, and uninstalling removes it. Keep your edited `otelcol.yaml` outside the installation directory, and after every upgrade run the two commands above again. Until you do, the Collector runs the stock configuration: it sends nothing to Netdata, and it listens for OTLP and Jaeger traffic on all network interfaces. Once your configuration is back, the receivers resume from their saved positions in `%ProgramData%`, so events logged in between are still ingested while the channels hold them.

   Alternatively, run the binary directly from an elevated prompt:

   ```powershell
   otelcol-contrib.exe --config .\otelcol.yaml
   ```

   Run this way, the Collector logs to the console, and you can enable the commented `debug` exporter to see every event it sends. Do not enable it for the service: a Windows service logs to the Application channel, so the debug output would flood that channel.

4. Generate a test event in the Application channel, from an elevated prompt:

   ```powershell
   eventcreate /T WARNING /ID 100 /L APPLICATION /SO otelcol-test /D "Test message for the Windows Event Log recipe"
   ```

5. Look at the ingested logs in the Netdata dashboard. A log record should have:

   - The rendered message as `log.body`, and the severity number and text (such as `Warning`) from the event level.
   - Resource attributes: `service.name` (the event provider, `otelcol-test` for the test event), `host.name` (the computer), and `process.pid` where the event records the process.
   - Log attributes `winlog.event_id`, `winlog.channel`, `winlog.provider_name`, `winlog.record_id` (the event record ID Event Viewer shows), and, where the event carries them, `winlog.task`, `winlog.opcode`, `winlog.keywords`, `winlog.provider_guid`, `winlog.activity_id` and `thread.id`.
   - For events logged for an account: `user.id` (its SID), `user.name` (the account name, such as `Administrator`), `winlog.user.domain` and `winlog.user.type`.
   - The event's payload as `winlog.event_data` (or `winlog.user_data`), with its named fields as nested keys, such as `winlog.event_data.TargetUserName`.

## 📡 Choosing channels

Each `windows_event_log` receiver reads one channel. Application and System are enabled.

- To add the Security channel, uncomment the `windows_event_log/security` receiver and its entry in the pipeline. Reading it requires administrative rights; the MSI service runs as `LocalSystem`, which has them. Security can produce a very large number of events, especially on domain controllers.
- To add any other channel, copy a receiver block, give it a new name after the slash (for example `windows_event_log/powershell`), set `channel` (for example `Microsoft-Windows-PowerShell/Operational`), and add it to the pipeline. List the channels on the host with `wevtutil el`.
- To filter events, or to read several channels with one receiver, use an [XML query](https://learn.microsoft.com/en-us/windows/win32/wes/consuming-events) instead of `channel`. The commented `windows_event_log/errors` receiver selects only critical and error events from Application and System. Event Viewer can generate the XML: create a custom view, then copy its XML tab.
- To exclude noisy providers from a channel, add them to `exclude_providers`. Every receiver that reads the Application channel, including queries that select it, must keep `otelcol-contrib` in that list: the Collector service logs there under that name, and ingesting its own logs would feed them back into the pipeline.

Each receiver resolves SIDs to account names (`resolve_sids`) through the Windows Local Security Authority, with a cache. Besides `user.name`, the receiver adds the resolved names next to the payload fields whose name ends in `Sid`, such as `winlog.event_data.TargetUserSid_Account`. It can only resolve accounts of the local system and of the domain the host belongs to.

## 🔒 Securing the connection

The recipe sends the events unencrypted, which is only suitable on a trusted network. To protect the connection, enable TLS on the Netdata Agent, and optionally mutual TLS so that only Collectors with a client certificate can send logs.

1. On the Netdata host, add a server certificate and key to the listener in `otel.yaml`. For mutual TLS, also add the CA that signed the client certificates:

   ```yaml
   receivers:
     otlp:
       protocols:
         grpc:
           endpoint: "0.0.0.0:4317"
           tls:
             cert_file: /etc/netdata/ssl/server-cert.pem
             key_file: /etc/netdata/ssl/server-key.pem
             # Mutual TLS: require a client certificate signed by this CA.
             client_ca_file: /etc/netdata/ssl/client-ca.pem
   ```

   The server certificate must be valid for the name or IP address the Collector uses in its `endpoint`. Restart the Agent. See the [OpenTelemetry Plugin documentation](https://github.com/netdata/netdata/blob/master/src/crates/otel-plugin/integrations/opentelemetry.md) for all options.

2. On the Windows host, replace the `tls` section of the `otlp` exporter. `ca_file` is the CA that signed the server certificate (omit it when the certificate is signed by a CA Windows already trusts). `cert_file` and `key_file` are the client certificate and key, for mutual TLS only:

   ```yaml
   tls:
     ca_file: C:/ProgramData/OpenTelemetry Collector/tls/ca.pem
     cert_file: C:/ProgramData/OpenTelemetry Collector/tls/client-cert.pem
     key_file: C:/ProgramData/OpenTelemetry Collector/tls/client-key.pem
   ```

   Keep the key readable only by administrators and `SYSTEM`. Restart the Collector.

When several Windows hosts send logs to the same Netdata Agent, a firewall rule that allows port 4317 only from those hosts is a good addition to either setup.

## 🍽️ Serving

- To tell these events apart from other OTLP sources on the node, filter on `scope.name` = `github.com/open-telemetry/opentelemetry-collector-contrib/receiver/windowseventlogreceiver`. Every event also carries `winlog.*` attributes.
- `service.name` is the event provider, a fixed name registered on the system (such as `Service Control Manager` or `Microsoft-Windows-Security-Auditing`), so the number of Netdata log streams stays bounded. Events without a provider name fall back to the channel name.
- Severity follows the Logs Data Model: `Critical` maps to ERROR2, `Error` to ERROR, `Warning` to WARN, `Information` to INFO, and `Verbose` to DEBUG. This also applies when Windows cannot provide the level name (for example with `suppress_rendering_info`), in which case the transform names the numeric level itself; level 0, used by classic event sources, becomes `Information`, as in Event Viewer. Provider-defined levels (16 and up) keep their number as severity text, with an unspecified severity. The receiver matches the English level names only. On Windows installations in another display language, the severity text is the localized name and the severity number stays unspecified.
- When the provider has no message for the event ID, the body becomes `<service.name> event <ID>`, and the event's details are in `winlog.event_data`.
- Delivery is at-least-once: the disk-backed sending queue buffers the backlog while the Netdata Agent is unreachable, and the retries never give up (`max_elapsed_time: 0`). The receiver moves its bookmark past each batch after handing it downstream, whether or not that succeeded, so a finite retry budget would discard events once it runs out (the default is 5 minutes). A collector crash or restart while events are in flight can still skip them. When the Collector stops while the Netdata Agent is unreachable, it logs `Exporting failed. Dropping data.` for the batches it was retrying; in testing, the disk-backed queue still delivered them after the restart.
- For a one-time backfill of everything a channel still holds, set `start_at: beginning` on its receiver. Expect a large replay and remove the setting afterwards.
- The Collector service logs to the Application channel under the `otelcol-contrib` provider, which the receivers exclude. Look for its messages in Event Viewer.
