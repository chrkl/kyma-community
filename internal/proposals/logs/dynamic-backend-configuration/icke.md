
kind: Pipeline
metadata:
  name: logging-loki
spec:
    type: logs
    elements:
      - type: filter
        content: |
            Name               modify
            Match              loki.*
            Add                 icke    test
      - type: output
        name: optional-name # for referencing the mistake in validation phase
        content: |
            Name               grafana-loki
            Alias              loki-output
            Match              loki.*
            Url                ${LOKI_ENDPOINT} # Defined in loki-credentials Secret
            LabelMapPath       /files/labelmap.json
      - type: file
        name: labelmap.json
        content: |
        {
            "kubernetes": {
              "namespace_name": "namespace",
              "pod_name": "pod"
            },
            "stream": "stream"
        }
    secretRefs:
    - name: loki-credentials
      namespace: default
---
kind: Pipeline
metadata:
  name: tracing-zipkin
spec:
    type: trace
    elements:
      - type: processor
        name: attributes/meta
        content:
            actions:
            - key: environment
                value: production
                action: insert
      - type: exporter
        name: zipkin/bla
        content:
            endpoint: http://zipkin.istio-system.svc.cluster.local:9411/api/v2/spans
            insecure: true
    secretRefs:
    - name: zipkin-credentials
      namespace: default
---
kind: Preset
apiVersion: telemetry.kyma-project.io/v1alpha1
metadata:
  name: CLS-logging
spec:
    type: logs
    elements:
      - type: output
        name: optional-name # for referencing the mistake in validation phase
        content: |
            Name               grafana-loki
            Alias              loki-output
            Match              loki.*
            Url                ${LOKI_ENDPOINT} # Defined in loki-credentials Secret
            LabelMapPath       /files/labelmap.json
      - type: file
        name: labelmap.json
        content: |
        {
            ...
            "stream": "stream"
        }
    placeholders:
      - name: LOKI_ENDPOINT
        description: endpoint URL
---
kind: PresetBinding
apiVersion: telemetry.kyma-project.io/v1alpha1
metadata:
  name: CLS-logging
spec:
    preset: CLS-logging
    secretRef: mySecret


    ?bindings:
      - placeholder: LOKI_ENDPOINT
        key: url


Discussion Points:
- mapping of bindings?
- yaml as syntax for processors
- filters versus processors. Keep filters for logging to distinguish different meaning and have easier migration later
- Having outputs detected allows us to limit the supported outputs and do not support other elements like [Service]
- different pipeline per type? Will require also LogPreset
  kind: LogPipeline
  metadata:
    name: logging-loki
  spec:
    filters:
      - content: |
            Name               modify
            Match              loki.*
            Add                 icke    test
    outputs:
      - name: optional-name # for referencing the mistake in validation phase
        content: |
            Name               grafana-loki
            Alias              loki-output
            Match              loki.*
            Url                ${LOKI_ENDPOINT} # Defined in loki-credentials Secret
            LabelMapPath       /files/labelmap.json
    files:
      - name: labelmap.json
        content: |
        {
            "kubernetes": {
              "namespace_name": "namespace",
              "pod_name": "pod"
            },
            "stream": "stream"
        }
    secretRefs:
    - name: loki-credentials
      namespace: default
      