# Vmalert Webhook Server (Slack)
When `Vmalert` conditions are met, this webhook queries logs from `VictoriaLogs` using `annotations.query` and sends an alert message to Slack.  
Since `VictoriaLogs` only returns numeric values, this webhook was created to extract log details. If `VictoriaLogs` supports this in the future, ~~it will no longer be needed.~~  
I've made a PR to integrate this feature into `vmalert`.  
[vmalert PR](https://github.com/VictoriaMetrics/VictoriaMetrics/pull/10070)  
[helm chart PR](https://github.com/VictoriaMetrics/helm-charts/pull/2583)  
Once this PR merged, you can enable it with flags  
`-notifier.vlogs.url`, `notifier.slack.url`, `notifier.vlogs.ingress` in vmalert
`server.notifier.webhookVlogs.enabled`,`server.notifier.webhookVlogs.vlogsURL`, `server.notifier.webhookVlogs.slackURL`, `server.notifier.webhookVlogs.ingressURL` in helm chart values.


# Alert Configuration
* When writing `Vmalert` alert rules, `annotations.query` **must** be specified. This query is used to fetch logs before sending alerts.  
Example:  
```yaml
- name: TEST
  type: vlogs
  rules:
    - alert: TEST-Alert
      expr: _time:3m * "error" | stats count() as err_cnt | filter err_cnt:>0
      for: 1m
      labels:
        severity: critical
        datasource: victoriaLogs
      annotations:
        description: 'Error Log Count: {{ .Value }}'
        query: '_time:3m * "error"' # the exact same query as `expr` but without `stats`
```
`expr` > `_time:3m * "error" | stats count() as err_cnt | filter err_cnt:>0`  
--> Checks for error logs within the last 3 minutes. Triggers an alert if more than 0 logs are found.  
`query` > `_time:3m * "error"`   
--> Fetches logs using this query and sends the corresponding logs to Slack.

# Local Testing

### Environment Variables
The webhook server retrieves endpoints from environment variables. You can use a tool like `direnv` or set them manually.  

#### Using `direnv`
1. Run `direnv allow`  
2. Create a `.envrc` file and run:  
   ```sh
   echo "dotenv" > .envrc
   ```
3. Create a `.env` file and define `VICTORIALOGS_ENDPOINT`, `SLACK_ENDPOINT`.  
   If you have an Ingress Endpoint of `VictoriaLogs`, also set `VLOG_INGRESS` in `.env` file.
   Example:  
   ```sh
   SLACK_ENDPOINT=https://my-endpoint.com
   ```


### Running Locally
Run the webhook server from the `src/` directory:  
```sh
go run .
```
Available test scripts:  
- `test.sh`: Sends an alert to webhook server

# Container Build
Build the image from the `src/` directory. 
- Build the image:  
  ```sh
  docker build -t <your image tag> .
  ```
- Push the image:  
  ```sh
  docker push <your image tag>
  ```

# Kubernetes Deployment
Since the endpoint URLs are provided as environment variables, update the relevant section in `webhook-deployment.yaml` as needed.  

#### `webhook-deployment.yaml`
```yaml
env:
  - name: VICTORIALOGS_ENDPOINT
    value: "http://victoria-logs-victoria-logs-single-server.victoria.svc.cluster.local:9428/select/logsql/query"
  - name: SLACK_ENDPOINT
    value: "https://hooks.slack.com/services/my-api"
  - name: VLOG_INGRESS
    value: "https://my-vlog.ingress.com"
```
Modify the values as needed.

# With Alertmanager
```yaml
receivers:
  - name: "vmalert-webhook"
  webhook_configs:
    - url: "http://vmalert-webhook.victoria.svc.cluster.local:8080/webhook"
      send_resolved: true
```
Modify the values as needed.
