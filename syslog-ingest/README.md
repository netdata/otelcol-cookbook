# 🍜 Recipe: Remote syslog Ingest

This recipe demonstrates ingesting (syslog) logs from remote entities, in particular network devices. The vast majority of devices emit their logs to a configured syslog server using the venerable [RFC 3164](https://datatracker.ietf.org/doc/rfc3164/) protocol, also known as BSD syslog.

The collector will listen for incoming log messages over UDP using the syslog receiver, normalizes the various properties according to the OpenTelemetry Semantic Conventions, then sends them to Netdata. When `add_attributes` is enabled, the receiver will add various attributes on the underlying network connection.

When using the [RFC 5424](https://datatracker.ietf.org/doc/rfc5424/) protocol instead, there is also support for its structured fields and message ID (log type classifier).
## 🧄 Ingredients

- OpenTelemetry Collector Contrib, see the main [README.md](../README.md) for instructions.
- The [`otelcol.yaml`](./otelcol.yaml) file from this directory.
- A device that emits syslog messages.


## 🥣 Preparation

1. Edit `otelcol.yaml` at the locations marked `TODO`:

   1. BSD syslog does not include any timezone information (or even a year 🙃). Therefore, you need to make sure you set the timezone to use for parsing the timestamp in the log line.
   2. The protocol to use. Unfortunately, the syslog receiver does not automatically detect which version of the protocol is used, so it must be set explicitly. The default of `rfc3164` is likely correct, but this is where to switch to `rfc5424` if that's supported by the log source.
   3. Optionally disable `add_attributes`.

2. Start the Collector Contrib binary using the provided configuration:

   ```sh
   otelcol-contrib --config ./otelcol.yaml
   ```

3. Configure the log source to send logs to the host/IP of this machine, using UDP on port 53514.


   In the absence of a device that emits syslog messages, you can also use the `logger(1)` command. The tag is the combination of the application name and optional process ID in brackets.


   ```sh
   echo "Test" | logger --tag otelcol-test[1234] --rfc3164 --udp --server 127.0.0.1 --port 53514
   ```

4. Look at the ingested logs in the Netdata dashboard. The logs should have:

- Resource attributes for the application name as `service.name`, the process ID as `process.pid`, the hostname in `host.name`.
- The timestamp taken from the original syslog message.
- The log message in `log.body`.
- The parsed severity and facility.
- The unparsed original syslog message in `log.attributes.log.record.original`
- The various attributes for the underlying UDP connection (when `add_attributes` is enabled).

## 🍽️ Serving

- If you have no need for RFC 5424 or added network attributes, you can use the simplified [`otelcol-simple.yaml`](./otelcol-simple.yaml).
- The [`otelcol-robust.yaml`](./otelcol-robust.yaml) adds a file-backed sending queue, to make it a little bit more robust.
