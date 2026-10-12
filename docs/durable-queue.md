# Durable Queue: Keeping Bursts from Overwhelming My Pi

I built this after the Wazuh → Qwen → iPhone workflow was already working. The problem was straightforward: Qwen runs locally on a small Raspberry Pi 5, and several high-severity alerts could arrive before one analysis finished. I wanted those alerts stored and processed sequentially, not a new fleet of model workers.

## At a Glance

<br>

<table align="center">
  <thead>
    <tr>
      <th>Decision</th>
      <th>Implemented behavior</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Admission</td>
      <td>Eligible Wazuh level 13–16 alerts; JSON validated before acceptance</td>
    </tr>
    <tr>
      <td>Durable storage</td>
      <td>Private SQLite database (WAL, full-sync), plus duplicate-event fingerprint</td>
    </tr>
    <tr>
      <td>Worker model</td>
      <td>One worker submitting to the existing local Qwen gateway</td>
    </tr>
    <tr>
      <td>Network boundary</td>
      <td>Wazuh manager → restricted SSH tunnel → loopback-only queue on the Pi</td>
    </tr>
    <tr>
      <td>Authentication</td>
      <td>Private token required on enqueue; no new LAN-facing queue port</td>
    </tr>
    <tr>
      <td>Thermal control</td>
      <td>Pause new job starts at 70 °C, resume at 62 °C, default cooldown between jobs. An inference already running is not interrupted.</td>
    </tr>
    <tr>
      <td>Response semantics</td>
      <td><code>202</code> = durably queued; <code>200</code> = previously accepted duplicate; not phone delivery</td>
    </tr>
    <tr>
      <td>Incident response</td>
      <td>Qwen remains advisory; no automatic blocking</td>
    </tr>
  </tbody>
</table>

<br>

## Where the Queue Fits

<table align="center">
  <tr>
    <td>
<pre>
Linux agents / UniFi
       |
       v
Wazuh detection + MITRE metadata (level >=13)
       |
       v
Custom integration on Wazuh manager
       |  token-authenticated HTTP over restricted SSH forwarding
       v
Pi localhost queue (SQLite / duplicate check / 1 worker)
       |
       v
Existing local Qwen gateway --> append-only audit
       |
       v
Existing ntfy publisher + optional threat intelligence --> iPhone
       |
       v
Human investigation and decision
</pre>
    </td>
  </tr>
</table>
