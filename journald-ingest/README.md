# 🍜 Recipe: Systemd Journal Ingest

This recipe ingests log messages from the [systemd journal](https://www.freedesktop.org/software/systemd/man/latest/systemd-journal.html), the central log repository on Linux systems running systemd. The journald receiver follows the journal using the `journalctl` binary, converts well-known journal fields to [OpenTelemetry Semantic Convention](https://opentelemetry.io/docs/specs/semconv/) attributes, and sends the log records to Netdata.

The `transform` processor completes the semantic-convention picture for fields the receiver itself leaves unprefixed-but-unmapped: it derives `service.name` from the systemd unit (falling back to the program name), maps `_MACHINE_ID`, `_UID`, `_SYSTEMD_CGROUP` and `_SYSTEMD_INVOCATION_ID` to `host.id`, `process.user.id`, `process.linux.cgroup` and `service.instance.id`, and drops per-entry bookkeeping attributes.

> [!NOTE]
> The journald receiver is at *alpha* stability, Linux-only, and `convert_to_semantic_conventions` requires OpenTelemetry Collector Contrib **v0.162.0 or newer**.

## 🧄 Ingredients

- OpenTelemetry Collector Contrib, see the main [README.md](../README.md) for instructions.
- The [`otelcol.yaml`](./otelcol.yaml) file from this directory.
- A Linux system with a systemd journal, and the `journalctl` binary (the receiver shells out to `journalctl --follow`).

## 🥣 Preparation

1. Check that the user which runs the Collector can read the journal (the receiver shells out to `journalctl`):

   ```sh
   sudo su -s /bin/bash -c 'journalctl --lines 5' <user>
   ```

   If that prints log lines, you are done — many setups grant read access already, for example to `wheel` and `adm` via ACLs that systemd's tmpfiles rules place on the journal directories. If it complains about permissions, grant read access. Journal files are owned by `root:systemd-journal` with mode `0640`, so membership of the group your distribution uses is usually enough:

   ```sh
   sudo usermod -aG systemd-journal <user>
   ```

   (or `adm`/`wheel`, depending on the distribution and its ACL setup). Log out and back in, or restart the service, so the new group is active, then re-run the check.

2. Edit `otelcol.yaml` at the locations marked `TODO`:

   1. Optionally restrict the ingest to specific units with `units`, or other journal fields with `matches`. Without filters, the whole journal (at `info` priority and above) is ingested.
   2. The two `file_storage` locations: the cursor (so restarts resume where they left off) and the on-disk sending queue. The defaults under `/tmp` are fine for a try-out; pick persistent directories for production.

3. Start the Collector using the provided configuration:

   ```sh
   otelcol-contrib --config ./otelcol.yaml
   ```

4. Generate a test message (this lands in the journal as a `user`-owned entry with `SYSLOG_IDENTIFIER=otelcol-test`):

   ```sh
   logger -t otelcol-test "Test message for the journald recipe"
   ```

   Prefer `logger -p daemon.warning ...` to also see the severity mapping, and watch entries from an actual unit, such as `sudo systemctl restart sshd`, to see unit-derived attributes.

5. Look at the ingested logs in the Netdata dashboard. A log record should have:

   - The `MESSAGE` field as `log.body`, and the severity (number and syslog level name) from `PRIORITY`.
   - Resource attributes: `service.name`, `host.name`, `host.id`, `process.pid`, `process.executable.path`, `process.executable.name`, `process.command_line`, `process.user.id`, `process.linux.cgroup`, and `service.instance.id` (the unit's invocation ID, unique per unit start).
   - Log attributes such as `syslog.facility.code`, `syslog.identifier`, and `code.file.path` where the sender provided them.
   - The remaining journal fields verbatim under `journald.*` attributes (the unit names are kept, so any derived attribute can be traced back).

## 🍽️ Serving

- `service.name` is only derived from *static* unit names: template instances (`polkit-agent-helper@1234-….service`) and generated scope units (`kitty-12345.scope`, `app-….scope`) embed per-session identifiers — PIDs, random suffixes — and would turn every login, terminal or crash into a new Netdata log stream. Entries from those units (and kernel messages) fall back to the program name reported in `SYSLOG_IDENTIFIER`. If you prefer unit names verbatim, drop the `IsMatch` guards and the `replace_pattern` statement in the transform.
- The receiver persists its journal cursor before handing entries downstream, so `retry_on_failure.enabled: true` is what keeps delivery at-least-once when the Netdata Agent is unreachable; keep it enabled unless losing those entries is acceptable.
- For a one-time backfill of everything since boot, set `start_at: beginning` — expect a large replay and remove the setting afterwards.
- Once you are happy, remove the `debug` exporter from the `exporters` section and from the pipeline.
