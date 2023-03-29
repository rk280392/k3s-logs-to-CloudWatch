# k3s-logs-to-CloudWatch
# How to stream k3s logs to CloudWatch using Rancher logging

### Requirements:

 - Gather k3s journal logs from each node in the cluster.
 - Parse the logs to forward only required fields.
 - Forward the parsed data to cloudwatch.

### Solution:

Rancher uses this logging operator https://kube-logging.github.io/docs/ that comes with below CRDS:
 
 - flow
 - clusterFlow
 - output
 - clusterOutput

You can read more about them here https://kube-logging.github.io/docs/configuration/

We will be using clusterFlow and clusterOutput as they are not namespaced.

clusterFlow defines a logging flow for Fluentd with filters and outputs. Using this we can define and apply filters to select only the desired data. Once parsed, data will be forwarded to clusterOutput object.

clusterOutput defines where to send the data. It supports several plugins https://banzaicloud.com/docs/one-eye/logging-operator/plugins/outputs/ but we will be using Cloudwatch. You can read the spec here https://banzaicloud.com/docs/one-eye/logging-operator/plugins/outputs/cloudwatch/.


Now we have clusterFlow to parse the data, and clusterOutput to define the destination of data. We need a way to get the journal logs from the nodes. HostTailer https://banzaicloud.com/docs/one-eye/logging-operator/configuration/crds/extensions/ is a CRD provided by https://banzaicloud.com/ supported on Rancher.

From the doc `HostTailer’s main goal is to tail custom files and transmit their changes to stdout. This way the logging-operator is able to process them`. An example usage is provided here https://banzaicloud.com/docs/one-eye/logging-operator/configuration/extensions/kubernetes-host-tailer/.

Similarly if you know the log file name, you can also use the file-tailer. 

The difference between the two is host-tailer looks at specific systemd service logs like k3s.service logs, while for file-tailer you need specify the exact location of the log file like /var/log/nginx/access.log. 

Here is the YAML to get the systemd journal logs from each host. This will create a daemonset. Pods will fetch the logs from the journal logs and output them to stdout.

`
apiVersion: logging-extensions.banzaicloud.io/v1alpha1
kind: HostTailer
metadata:
  name: k3s-systemd-tailer
  namespace: cattle-logging-system
spec:
  systemdTailers:
    - name: k3s-systemd-tailer
      maxEntries: 100
      path: /run/log/journal/
      systemdFilter: k3s.service
`

The log output will be then fed to clusterFlow, which will parse the logs.

`
apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterFlow
metadata:
  name: host-tailer-flow
  namespace: cattle-logging-system
spec:
  filters:
    - parser:
        key_name: message
        reserve_time: true
        parse:
          type: json
    - record_transformer:
        remove_keys: _CMDLINE,_BOOT_ID,_MACHINE_ID,PRIORITY,SYSLOG_FACILITY,_UID,_GID,_SELINUX_CONTEXT,_SYSTEMD_SLICE,_CAP_EFFECTIVE,_TRANSPORT,_SYSTEMD_CGROUP,_SYSTEMD_INVOCATION_ID,_STREAM_ID,SYSLOG_IDENTIFIER,_COMM,_EXE
  match:
    - select: 
        labels:
          app.kubernetes.io/name: host-tailer
  globalOutputRefs:
    - host-logging-cloudwatch
`

Here we are matching the app name to the name of host tailer daemonset which is host-tailer. Once matched, we are parsing them using 'parser' https://docs.fluentd.org/filter/parser plugin. We only need the message field from the logs, so key_name is specified as 'message' and parse type is set to json to get the ouptut in json.

After this from the message field we are removing unwanted fields using remove_keys spec from the record_transformer https://docs.fluentd.org/filter/record_transformer plugin.

The globalOutputRefs is set to the name of the clusterOutput. 

`

apiVersion: logging.banzaicloud.io/v1beta1
kind: ClusterOutput
metadata:
  name: host-logging-cloudwatch
  namespace: cattle-logging-system
spec:
  cloudwatch:
    auto_create_stream: true
    format:
      type: json
    buffer:
      timekey: 30s
      timekey_use_utc: true
      timekey_wait: 30s
    log_group_name: hosted-group
    log_stream_name: host-logs
    region: aws-region

`

In the clusterOutput spec we are using cloudwatch with log_group_name, log_stream_name, region values passed a variable.
