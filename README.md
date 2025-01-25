# Tuskr Helm Chart
Launch jobs on k8s with HTTP.

1. Clone this repo
2. `helm install tuskr . -n tuskr --create-namespace`
3. Create a job template to the namespace you want.
```
apiVersion: tuskr.io/v1alpha1
kind: JobTemplate
metadata:
  name: tuskr-test-job-template
  namespace: default
spec:
  jobSpec:
    template:
      metadata:
        labels:
          app: tuskr-test-job
      spec:
        restartPolicy: Never
        containers:
          - name: main
            image: busybox
            command: ["/bin/sh"]
            args: ["-c", "echo Hello from the template && sleep 30"]
```
4. `kubectl apply -f job-template.yaml`
5. From any container in the cluster, send a POST request to the tuskr service.
```
curl -X POST http://tuskr-controller.tuskr.svc.cluster.local:8080/launch \
  -H "Content-Type: application/json" \
  -d '{
    "jobTemplate": {
      "name": "tuskr-test-job-template",
      "namespace": "default"
    }
  }'
```
