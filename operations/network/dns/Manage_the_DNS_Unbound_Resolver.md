# Manage the DNS Unbound Resolver

The unbound DNS instance is used to resolve names for the physical equipment on the management networks within the system, such as NCNs, UANs, switches, and compute nodes.
This instance is accessible only within the HPE Cray EX system.

- [Check the status of the `cray-dns-unbound` pods](#check-the-status-of-the-cray-dns-unbound-pods)
- [Unbound logs](#unbound-logs)
- [View manager (DNS Helper) logs](#view-manager-dns-helper-logs)
- [Restart Unbound](#restart-unbound)
- [Clear bad data in the Unbound ConfigMap](#clear-bad-data-in-the-unbound-configmap)
- [Change the site DNS server](#change-the-site-dns-server)
- [Increase the number of Unbound pods](#increase-the-number-of-unbound-pods)
- [Change which HSN NIC is used for the node alias](#change-which-hsn-nic-is-used-for-the-node-alias)
- [Create custom DNS records](#create-custom-dns-records)

## Check the status of the `cray-dns-unbound` pods

(`ncn-mw#`) Check the status of the pods:

```bash
kubectl get -n services pods | grep unbound
```

Example output:

```text
cray-dns-unbound-696c58647f-26k4c            2/2   Running      0   121m
cray-dns-unbound-696c58647f-rv8h6            2/2   Running      0   121m
cray-dns-unbound-coredns-q9lbg               0/2   Completed    0   121m
cray-dns-unbound-manager-1596149400-5rqxd    0/2   Completed    0   20h
cray-dns-unbound-manager-1596149400-8ppv4    0/2   Completed    0   20h
cray-dns-unbound-manager-1596149400-cwksv    0/2   Completed    0   20h
cray-dns-unbound-manager-1596149400-dtm9p    0/2   Completed    0   20h
cray-dns-unbound-manager-1596149400-hckmp    0/2   Completed    0   20h
cray-dns-unbound-manager-1596149400-t24w6    0/2   Completed    0   20h
cray-dns-unbound-manager-1596149400-vzxnp    0/2   Completed    0   20h
cray-dns-unbound-manager-1596222000-bcsk7    0/2   Completed    0   2m48s
cray-dns-unbound-manager-1596222060-8pjx6    0/2   Completed    0   118s
cray-dns-unbound-manager-1596222120-hrgbr    0/2   Completed    0   67s
cray-dns-unbound-manager-1596222180-sf46q    1/2   NotReady     0   7s
```

For more information about the pods displayed in the output above:

- `cray-dns-unbound-xxx` - These are the main unbound pods.
- `cray-dns-unbound-manager-yyy` - These are job pods that run periodically to update DNS from DHCP \(Kea\) and the SLS/SMD content for the Hardware State Manager \(HSM\).
  Pods will go into the `Completed` status, and then independently be reaped later by Kubernetes.
- `cray-dns-unbound-coredns-zzz` - This pod is run one time during installation of Unbound and reconfigures CoreDNS/ExternalDNS to point to Unbound for all site/internet lookups.

The table below describes what the status of each pod means for the health of the `cray-dns-unbound` services and pods. The `Init` and `NotReady` states are not necessarily bad;
they mean that the pod is being started or is processing. The `cray-dns-manager` and `cray-dns-coredns` pods for `cray-dns-unbound` are job pods that run periodically.

| Pod                | Healthy Status | Error Status       | Other          |
|--------------------|----------------|--------------------|----------------|
| `cray-dns-unbound` | `Running`      | `CrashBackOffLoop` |                |
| `cray-dns-coredns` | `Completed`    | `CrashBackOffLoop` | `InitNotReady` |
| `cray-dns-manager` | `Completed`    | `CrashBackOffLoop` | `InitNotReady` |

## Unbound logs

(`ncn-mw#`) Logs for the Unbound pods will show the status and health of actual DNS lookups.
Any logs with `ERROR` or `Exception` are an indication that the Unbound service is not healthy.

```bash
kubectl logs -n services -l app.kubernetes.io/instance=cray-dns-unbound -c unbound
```

Example output:

```text
[1596224129] unbound[8:0] debug: using localzone health.check.unbound. transparent
[1596224129] unbound[8:0] debug: using localzone health.check.unbound. transparent
[1596224135] unbound[8:0] debug: using localzone health.check.unbound. transparent
[1596224135] unbound[8:0] debug: using localzone health.check.unbound. transparent
[1596224140] unbound[8:0] debug: using localzone health.check.unbound. transparent
[1596224140] unbound[8:0] debug: using localzone health.check.unbound. transparent
[1596224145] unbound[8:0] debug: using localzone health.check.unbound. transparent
[1596224145] unbound[8:0] debug: using localzone health.check.unbound. transparent
[1596224149] unbound[8:0] debug: using localzone health.check.unbound. transparent
[1596224149] unbound[8:0] debug: using localzone health.check.unbound. transparent
...snip...
[1597020669] unbound[8:0] error: error parsing local-data at 33 '69.0.254.10.in-addr.arpa.  PTR  .local': Empty label
[1597020669] unbound[8:0] error: Bad local-data RR 69.0.254.10.in-addr.arpa.  PTR  .local
[1597020669] unbound[8:0] fatal error: Could not set up local zones
```

### Troubleshooting Unbound log errors

If there are any errors in the Unbound logs:

- The `using localzone health.check.unbound. transparent` log is not an issue.
- Typically, any error seen in Unbound, including the example above, falls under one of two categories:
    - A bad configuration from a misconfiguration in the Helm chart. Currently, only the site/external DNS lookup can be at fault.
        - **ACTION:** See the `customization.yaml` file and look at the `system_to_site_lookup` values. Ensure that the external lookup values are valid and working.
    - Bad data \(as shown in the above example\) comes only from the DNS Helper and can be seen in the manager logs.
        - **ACTION:** See [View manager (DNS Helper) logs](#view-manager-dns-helper-logs).

## View manager \(DNS Helper\) logs

Manager logs will show the status of the latest "true up" of DNS with respect to DHCP actual leases and SLS/SMD status.

(`ncn-mw#`) The following command shows the last four lines of the last manager run, and can be adjusted as needed.

```bash
kubectl logs -n services pod/$(kubectl get -n services pods | grep unbound | tail -n 1 | cut -f 1 -d ' ') -c manager | tail -n4
```

Example output:

```text
uid: bc1e8b7f-39e2-49e5-b586-2028953d2940

Comparing new and existing DNS records.
    No differences found. Skipping DNS update
```

Any log with `ERROR` or `Exception` is an indication that DNS is not healthy. The above example includes one of two possible reports for a healthy manager run.
The healthy states are described below, as long as the write to the ConfigMap has not failed:

- `No differences found. Skipping DNS update`
- `Differences found. Writing new DNS records to our configmap.`

### Troubleshooting the manager

The manager runs periodically, about once every minute. Check if this is a one-time occurrence or if it is a recurring issue.

- If the error shows in one manager log, but not during the next one, then this is likely a one-time failure.
    - Check to see if the record exists in DNS, and if so, move on.
- If several or all manager logs show errors, particularly the same error, then this could be one of several sources:
    - Bad network connections to DHCP or SLS/SMD.
        - **ACTION:** Capture as much log data as possible and contact customer support.
    - Bad data from DHCP or SLS/SMD.
        - **ACTION:** If connections to DHCP \(Kea\) are involved, then refer to [Troubleshoot DHCP Issues](../dhcp/Troubleshoot_DHCP_Issues.md).

## Restart Unbound

If any errors discovered in the sections above have been deemed transient or have not been resolved, then restart the Unbound pods.

(`ncn-mw#`) Use the following command to restart the pods:

```bash
kubectl -n services rollout restart deployment cray-dns-unbound
```

A rolling restart of the Unbound pods will occur; old pods will not be terminated and new pods will not be added to the load balancer until the new pods have successfully loaded the DNS records.

## Clear bad data in the Unbound ConfigMap

Unbound stores records it obtains from DHCP, SLS, and SMD via the manager job in a ConfigMap. It is possible to clear this ConfigMap and allow the next manager job to regenerate the content.

This is useful in the following cases:

- A transient failure in any Unbound process or required services has left the configuration data in a bad state.
- SLS and SMD data needed to be reset because of bad or incorrect data there.
- DHCP \(Kea\) has been restarted to clear errors.

(`ncn-mw#`) The following clears the \(DNS Helper\) manager generated data in the ConfigMap. This is generally safe as Unbound runtime data is held elsewhere.

```bash
kubectl -n services patch configmaps cray-dns-unbound --type merge -p '{"binaryData":{"records.json.gz":"H4sICLQ/Z2AAA3JlY29yZHMuanNvbgCLjuUCAETSaHADAAAA"}}'
```

## Change the site DNS server

Use the following procedure to change the site DNS server that Unbound forwards queries to. This may be necessary if the site DNS server is moved to a different IP address.

1. (`ncn-mw#`) Edit the `cray-dns-unbound` ConfigMap.

   ```bash
   kubectl -n services edit configmap cray-dns-unbound
   ```

   Update the `forward-zone` value in `unbound.conf`.

   ```yaml
   forward-zone:
       name: .
       forward-addr: 172.30.84.40
   ```

   Multiple DNS servers can be defined if required.

   ```yaml
   forward-zone:
       name: .
       forward-addr: 172.30.84.40
       forward-addr: 192.168.0.1
   ```

1. (`ncn-mw#`) Restart `cray-dns-unbound` for this change to take effect.

   ```bash
   kubectl -n services rollout restart deployment cray-dns-unbound
   ```

   Example output:

   ```text
   deployment.apps/cray-dns-unbound restarted
   ```

1. (`ncn-mw#`) Update `customizations.yaml`.

   **IMPORTANT:** If this step is not performed, then the Unbound configuration will be overwritten with the previous value the next time CSM or Unbound is upgraded.

   1. Extract `customizations.yaml` from the `site-init` secret in the `loftsman` namespace.

      ```bash
      kubectl -n loftsman get secret site-init -o json | jq -r '.data."customizations.yaml"' | base64 -d > customizations.yaml
      ```

   1. Update `system_to_site_lookups` with the value of the new DNS server.

      ```yaml
      spec:
        network:
          netstaticips:
            system_to_site_lookups: 172.30.84.40
      ```

      If multiple DNS servers are required, add the additional servers into the `cray-dns-unbound` service configuration.

      ```yaml
      spec:
        kubernetes:
          services:
            cray-dns-unbound:
              forwardZones:
                - name: "."
                  forwardIps:
                    - "{{ network.netstaticips.system_to_site_lookups }}"
                    - "192.168.0.1"
              domain_name: '{{ network.dns.external }}'
      ```

   1. Update the `site-init` secret in the `loftsman` namespace.

      ```bash
      kubectl delete secret -n loftsman site-init
      kubectl create secret -n loftsman generic site-init --from-file=customizations.yaml
      ```

## Increase the number of Unbound pods

On large systems it may be necessary to increase the number of Unbound Pods because of the increased DNS query load.
See [Scale `cray-dns-unbound` service](../../CSM_product_management/Post_Install_Customizations.md#scale-cray-dns-unbound-service) for more information.

## Change which HSN NIC is used for the node alias

Previous CSM versions associate all HSN IPs with the node `nid` alias.

Example output from the `host nid000001` command:

```text
nid000001 has address 10.253.0.1
nid000001 has address 10.253.0.1
nid000001 has address 10.253.0.1
nid000001 has address 10.253.0.1
```

Some workload managers do not handle this well so CSM 1.6 and above will only use the IP address of the first HSN NIC for this alias.

Example output from the `host nid000001` command:

```text
nid000001 has address 10.253.0.1
```

This behaviour is configurable, use the following procedure to change the HSN NIC used for the `nid` alias.

1. (`ncn-mw#`) Edit the `cray-dns-unbound-manager` CronJob.

   ```bash
   kubectl -n services edit cronjob cray-dns-unbound-manager
   ```

   Update the `HSN_NIC_ALIAS` environment variable in the CronJob with the desired value.

   Valid values are "all" or the numeric index of a HSN interface. This variable should be set to the number of an interface that is common to all nodes with HSN interfaces.
   For example if `HSN_NIC_ALIAS` is set to `4` and there are nodes in the system that only have two HSN interfaces then aliases will not be created for those nodes.

   ```yaml
   - name: HSN_NIC_ALIAS
     value: "all"
   ```

1. (`ncn-mw#`) Update `customizations.yaml`.

   **IMPORTANT:** If this step is not performed, then the Unbound configuration will be overwritten with the previous value the next time CSM or Unbound is upgraded.

   1. Extract `customizations.yaml` from the `site-init` secret in the `loftsman` namespace.

      ```bash
      kubectl -n loftsman get secret site-init -o json | jq -r '.data."customizations.yaml"' | base64 -d > customizations.yaml
      ```

   1. Update `hsnNicAlias` with the desired value.

      ```bash
      yq w -i customizations.yaml spec.kubernetes.services.cray-dns-unbound.hsnNicAlias all
      ```

   1. Update the `site-init` secret in the `loftsman` namespace.

      ```bash
      kubectl delete secret -n loftsman site-init
      kubectl create secret -n loftsman generic site-init --from-file=customizations.yaml
      ```

## Create custom DNS records

It may be desirable to add custom records to the DNS service, for example a "service" (`SRV`) record for a Workload Manager to allow clients to determine the host
and port of this service. This can be achieved by adding `local-data` or `local-data-ptr` records to the `custom_records.conf` key of the `cray-dns-unbound` ConfigMap.

The following example will create a `SRV` record for the `slurm-host` service in the `local` domain which uses the `TCP` protocol, port `6817`,  and will point to the
existing `slurmctld-service.local` record.

A `SRV` record has the following format:

```text
_service._protocol.name. TTL class type of record priority weight port target.
```

So the `slurm-host` record would be formatted as:

```text
_slurm-host._tcp.local. 3600 IN SRV 10 0 6817 slurmctld-service.local.
```

To add this record to the DNS configuration:

1. (`ncn-mw#`) Add the record to the `cray-dns-unbound` ConfigMap.

   1. Edit the `cray-dns-unbound` ConfigMap.

      ```bash
      kubectl -n services edit cm cray-dns-unbound
      ```

   2. Add the record to the `custom_records.conf` key of the ConfigMap.

      **IMPORTANT:** Syntax errors in `custom_records.conf` may cause the DNS service to fail if the configuration cannot be parsed by the server.

      Example output:

      ```yaml
      data:
        custom_records.conf: |-
          # Add any additional local-data or local-data-ptr records here, one per line.
          # See https://unbound.docs.nlnetlabs.nl/en/latest/manpages/unbound.conf.html#unbound-conf-local-data for syntax.
          # WARNING: Syntax errors here will cause Unbound to fail to start and the cluster DNS service will fail.
          #
          # Examples:
          # local-data: "axfr-service.example.com. A 10.252.4.254"
          # local-data-ptr: "10.252.4.254 axfr-service.example.com"
          # local-data: "_axfr-service._tcp.example.com. 3600 IN SRV 0 100 8080 axfr-service.example.com."
          local-data: "_slurm-host._tcp.local. 3600 IN SRV 0 100 6818 slurmctld-service.local."
      ```

      Once saved, the `cray-dns-unbound` service will automatically pick up the record on the next refresh cycle. It may take up to 90 seconds for all replicas of the `cray-dns-unbound` service to be consistent.

1. (`ncn-mw#`) Query the DNS service to verify the record was created.

   ```bash
   host -t SRV _slurm-host._tcp.local. 10.92.100.225
   ```

   Example output:

   ```text
   Using domain server:
   Name: 10.92.100.225
   Address: 10.92.100.225#53
   Aliases:

   _slurm-host._tcp.local has SRV record 0 100 6818 slurmctld-service.local.
   ```

See the [Unbound DNS documentation](https://unbound.docs.nlnetlabs.nl/en/latest/manpages/unbound.conf.html#unbound-conf-local-data) for more information about `local-data` records.

Custom records added using this procedure will persist through a CSM upgrade or an upgrade of the `cray-dns-unbound` Helm chart but will be removed if the `cray-dns-unbound`
Helm chart is uninstalled and reinstalled. A backup of the records can be taken with the following command.

```bash
kubectl -n services get cm cray-dns-unbound -o yaml | yq4 '.data."custom_records.conf"'
```

Example output:

```yaml
# Add any additional local-data or local-data-ptr records here, one per line.
# See https://unbound.docs.nlnetlabs.nl/en/latest/manpages/unbound.conf.html#unbound-conf-local-data for syntax.
# WARNING: Syntax errors here will cause Unbound to fail to start and the cluster DNS service will fail.
#
# Examples:
# local-data: "axfr-service.example.com. A 10.252.4.254"
# local-data-ptr: "10.252.4.254 axfr-service.example.com"
# local-data: "_axfr-service._tcp.example.com. 3600 IN SRV 0 100 8080 axfr-service.example.com."
local-data: "_slurm-host._tcp.local. 3600 IN SRV 0 100 6818 slurmctld-service.local."
```

If the `cray-dns-unbound` Helm chart is reinstalled then the records can be restored by following this procedure again.
