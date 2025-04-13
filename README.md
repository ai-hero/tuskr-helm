# Tuskr Helm Chart
Launch jobs on k8s with HTTP.

1. Clone this repo
2. `helm install tuskr . -n tuskr --create-namespace` or `helm upgrade tuskr . -n tuskr` to upgrade.
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
6. Here's how you can delete tuskr:
```
helm uninstall tuskr -n tuskr
```

## Full Example
```python
import requests
import time
import json
from typing import Tuple, Dict, Optional


class JobMonitoringError(Exception):
    """Custom exception for job monitoring errors"""

    pass


def launch_job() -> Dict:
    """
    Launch a new job and return the response data.
    Configures the job to cat the contents of hello_world.txt from the mounted inputs directory.
    """
    url = "http://localhost:8080/launch"
    payload = {
        "jobTemplate": {"name": "tuskr-test-job-template", "namespace": "default"},
        "command": ["cat"],  # Use cat command
        "args": ["/mnt/data/inputs/hello_world.txt"],  # Path to the mounted input file
        "inputs": {
            "hello_world.txt": "Hello, World!\n"  # Content of the input file
        },
    }
    headers = {"Content-Type": "application/json"}

    try:
        response = requests.post(url, json=payload, headers=headers)
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        raise JobMonitoringError(f"Failed to launch job: {str(e)}")


def get_pod_status(pod_info: Dict) -> Tuple[str, str]:
    """
    Extract pod status and detailed message from pod information

    Returns:
        Tuple containing status string and detailed message
    """
    if not pod_info:
        return "Unknown", "No pod information available"

    status = pod_info.get("status", {})
    phase = status.get("phase", "Unknown")

    # Check container statuses
    container_statuses = status.get("container_statuses", [])
    if container_statuses:
        container = container_statuses[0]  # Get the main container
        state = container.get("state", {})

        if state.get("waiting"):
            reason = state["waiting"].get("reason", "Unknown")
            message = state["waiting"].get("message", "Container is waiting")
            return reason, message
        elif state.get("running"):
            return "Running", "Container is running"
        elif state.get("terminated"):
            exit_code = state["terminated"].get("exitCode", -1)
            reason = state["terminated"].get("reason", "Unknown")
            message = state["terminated"].get("message", "Container terminated")
            return f"Terminated ({reason})", f"Exit code: {exit_code}, {message}"

    # Check pod conditions
    conditions = status.get("conditions", [])
    messages = []
    for condition in conditions:
        if condition.get("status") == "False" and condition.get("message"):
            messages.append(f"{condition['type']}: {condition['message']}")

    if messages:
        return phase, " | ".join(messages)

    return phase, "Pod status unknown"


def get_job_status(job_info: Dict) -> Tuple[bool, str, str]:
    """
    Get comprehensive job status including pod details

    Returns:
        Tuple containing:
        - Boolean indicating if job is completed
        - Status string
        - Detailed status message
    """
    if not job_info:
        return False, "Unknown", "No job information available"

    status = job_info.get("status", {})

    # Check if job is succeeded
    if status.get("succeeded"):
        return True, "Complete", "Job completed successfully"

    # Check if job failed
    if status.get("failed"):
        return True, "Failed", f"Job failed: {status.get('failed')} pods failed"

    # Check active status
    active = status.get("active", 0)
    if active > 0:
        return False, "Running", f"Job is running with {active} active pod(s)"

    # Check conditions
    conditions = status.get("conditions", [])
    for condition in conditions:
        condition_type = condition.get("type")
        if condition_type == "Failed" and condition.get("status") == "True":
            return True, "Failed", condition.get("message", "Job failed")
        elif condition_type == "Complete" and condition.get("status") == "True":
            return True, "Complete", condition.get("message", "Job completed")

    # If no definitive status found, look at pod status
    pods = job_info.get("pods", [])
    if pods:
        pod_status, pod_message = get_pod_status(pods[0])
        return False, pod_status, pod_message

    return False, "Pending", "Job is pending or initializing"


def get_and_print_logs(base_url):
    """Retrieve and display logs in a formatted way."""
    logs_response = requests.get(f"{base_url}/logs")
    if logs_response.status_code == 200:
        log_data = logs_response.json()
        print(f"\nJob Logs for {log_data['job']} in namespace {log_data['namespace']}:")
        print("=" * 80)

        for entry in log_data["logs"]:
            # Print header for each pod/container
            print(f"\nPod: {entry['pod_name']}")
            print(f"Container: {entry['container_name']}")
            print("-" * 40)

            if "logs" in entry:
                # If logs exist, print them with proper formatting
                log_content = entry["logs"].strip()
                if log_content:
                    print(log_content)
                else:
                    print("(No logs available)")
            else:
                # If there was an error getting logs, print the error
                print(f"Error retrieving logs: {entry.get('error', 'Unknown error')}")

            print("-" * 40)
    else:
        print(f"\n--- Failed to get logs: ({logs_response.status_code})")
        print(f"Response: {logs_response.text}")


def get_job_info(namespace: str, job_name: str) -> Optional[Dict]:
    """
    Get comprehensive job information including logs and status
    """
    base_url = f"http://localhost:8080/jobs/{namespace}/{job_name}"

    try:
        # Get basic job info first
        job_response = requests.get(base_url)
        job_response.raise_for_status()
        job_info = job_response.json()

        # Get job description for events and pod details
        describe_response = requests.get(f"{base_url}/describe")
        if describe_response.status_code == 200:
            describe_info = describe_response.json()
            # Add pods info to job_info for status checking
            job_info["pods"] = describe_info.get("pods", [])

        # Get job logs
        get_and_print_logs(base_url)

        return job_info

    except requests.exceptions.RequestException as e:
        print(f"Error getting job info: {str(e)}")
        return None


def monitor_job(timeout_seconds: int = 300) -> None:
    """
    Launch and monitor a job until completion or timeout
    """
    try:
        # Launch the job
        result = launch_job()
        print(f"Job launched: {result}")

        # Extract job name and namespace
        message = result.get("message", "")
        if not message:
            raise JobMonitoringError("No job message received")

        try:
            job_name = message.split("'")[1]
            namespace = message.split("'")[3]
        except IndexError:
            raise JobMonitoringError(
                f"Failed to parse job info from message: {message}"
            )

        start_time = time.time()
        last_status = None
        last_message = None

        # Poll for job status
        while True:
            if time.time() - start_time > timeout_seconds:
                raise JobMonitoringError(
                    f"Job monitoring timed out after {timeout_seconds} seconds"
                )

            job_info = get_job_info(namespace, job_name)
            if not job_info:
                print("Failed to get job info, retrying...")
                time.sleep(5)
                continue

            completed, status, message = get_job_status(job_info)

            # Only print status if it changed
            if status != last_status or message != last_message:
                print(f"\nJob Status: {status}")
                print(f"Status Details: {message}")
                last_status = status
                last_message = message

            if completed:
                print(f"\nJob {job_name} finished with status: {status}")
                if status == "Failed":
                    raise JobMonitoringError(f"Job failed: {message}")
                break

            time.sleep(5)

    except JobMonitoringError as e:
        print(f"Job monitoring error: {str(e)}")
        raise
    except Exception as e:
        print(f"Unexpected error: {str(e)}")
        raise


if __name__ == "__main__":
    try:
        monitor_job()
    except Exception as e:
        print(f"Failed to monitor job: {str(e)}")
        exit(1)
```
