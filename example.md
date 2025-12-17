# Gatekeeper Mutation with External Data for OCM Placement

This is just brainstorming and not tested at all!!!

Goal is to modify a Placement using Gatekeeper-Mutation, at the end the python script should query metrics from the RHACM-Hub Cluster.

### 1. Python Provider Script (`server.py`)
Run this service where Gatekeeper can reach it (e.g., containerize it and run as a Service).

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/update-gpu-score', methods=['POST'])
def update_gpu_score():
    # 1. Parse the incoming request from Gatekeeper
    request_data = request.json
    results = []

    # Gatekeeper sends a list of 'keys'.
    # Because 'dataSource: ValueAtLocation' is set on 'status.scores', 
    # the key IS the list of score objects.
    for scores_list in request_data.get("request", {}).get("keys", []):
        
        # Handle cases where the list is None (newly created object)
        current_scores = scores_list if scores_list else []
        found_target = False
        
        # 2. Iterate through the existing scores
        for item in current_scores:
            if item.get("name") == "free-capacity":
                # LOGIC: Update the value here (e.g. from your DB/Metrics)
                item["value"] = 88 
                found_target = True

        # 3. If the metric is missing, append it
        if not found_target:
            current_scores.append({
                "name": "free-capacity",
                "value": 88
            })

        # 4. Format the response
        # We must return the 'key' (original input) and 'value' (new output)
        results.append({
            "key": scores_list,    
            "value": current_scores 
        })

    # 5. Return the standard Gatekeeper ProviderResponse
    return jsonify({
        "apiVersion": "externaldata.gatekeeper.sh/v1beta1",
        "kind": "ProviderResponse",
        "response": {
            "idempotent": True, # Required for mutations
            "items": results
        }
    })

if __name__ == '__main__':
    # SSL is required by Gatekeeper. Use 'adhoc' for local dev only.
    app.run(host='0.0.0.0', port=8443, ssl_context='adhoc')
```

---

### 2. Gatekeeper Configuration (`gatekeeper-config.yaml`)
Apply this to the cluster.

```yaml
# ---------------------------------------------------------
# 1. Register the Python Service as a Provider
# ---------------------------------------------------------
apiVersion: externaldata.gatekeeper.sh/v1beta1
kind: Provider
metadata:
  name: gpu-score-provider
spec:
  # Replace with the actual DNS name/IP of your python service
  url: [https://gpu-service.default.svc:8443/update-gpu-score](https://gpu-service.default.svc:8443/update-gpu-score)
  
  timeout: 3
  
  # CA Bundle is mandatory. Use `base64 encoded` certificate.
  # For quick testing, you can use a generated cert bundle here.
  caBundle: "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0t..." 

---
# ---------------------------------------------------------
# 2. Define the Mutation Policy
# ---------------------------------------------------------
apiVersion: mutations.gatekeeper.sh/v1beta1
kind: Assign
metadata:
  name: update-addon-score
spec:
  applyTo:
  - groups: ["cluster.open-cluster-management.io"]
    kinds: ["AddOnPlacementScore"]
    versions: ["v1alpha1"]

  # We want to Read AND Write to the scores list
  location: "status.scores"

  parameters:
    assign:
      externalData:
        provider: gpu-score-provider
        # This sends the actual JSON list at 'location' to your Python script
        dataSource: ValueAtLocation 
        failurePolicy: Ignore # Use 'Fail' if strict enforcement is needed

  # Only apply this to specific objects/namespaces
  match:
    scope: Namespaced
    namespaces: ["cluster1"]
```

---

### 3. Test Object (`test-score.yaml`)
Apply this to verify the mutation works.

```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: AddOnPlacementScore
metadata:
  name: gpu-usage-score
  namespace: cluster1
status:
  scores:
    # We provide a dummy value of 8.
    # After applying, if you run 'kubectl get', this should become 88.
    - name: free-capacity
      value: 8
  validUntil: "2024-12-31T10:00:00Z"
```


