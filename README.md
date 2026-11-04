- st2 key set work_dir "<dir>"
- st2 key set ssh_username <user>
- st2 key set ssh_password <pass> --encrypt

## Proxmox web-tier scaling

The `scaling_pve.on_request` workflow is triggered by the `/api/v1/webhooks/scale`
webhook. It reads the Terraform `total_machines` output, changes the count by one
when the requested direction is within the configured bounds, and then updates
the HAProxy runtime socket. If `total_machines` is supplied, it is used as an
absolute target; otherwise the workflow changes the count by one.

The Terraform directory must expose `total_machines` and a list-valued
`worker_ips` output (or set `terraform_ip_output` on the action). The remote
hosts must have Terraform, `jq`, and the HAProxy `socat` command available.

```bash
curl -X POST https://<st2-ip>/api/v1/webhooks/scale \
  -H "St2-Api-Key: <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"direction":"scale_out","total_machines":5,"terraform_host":"terraform-host","lb_host":"lb-host"}'
```

Use `direction: scale_in` with `total_machines` to scale in to an absolute
count, for example `{"direction":"scale_in","total_machines":2}`. Omitting
`total_machines` removes or adds one VM. Requests with `auto` or requests
outside `min_machines` / `max_machines` complete without changing capacity.
