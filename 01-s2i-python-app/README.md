# Demo 1: Python Source-to-Image (S2I) on OpenShift

This demo shows how to deploy a production-grade Python web application onto Red Hat OpenShift using the official Red Hat Universal Base Image (UBI) Python S2I builder.

---

## 1. What This Demo Contains

- **`app.py`**: Robust WSGI application compliant with Python 3.9+ and Gunicorn. Exposes health endpoints (`/healthz`, `/readyz`) and runtime diagnostics (`/info`).
- **`requirements.txt`**: Standard dependencies automatically processed during `s2i assemble`.
- **`.s2i/environment`**: S2I builder configuration variables (`APP_FILE`, `APP_MODULE`, `PORT=8080`).
- **`k8s/`**: Declarative OpenShift manifests (`ImageStream`, `BuildConfig`, `Deployment`, `Service`, `Route`).
- **`test_app.py`**: Zero-dependency unit and contract test suite.

---

## 2. Red Hat S2I Builder Reference

This demo uses the official Red Hat UBI 9 Python S2I builder:
- **ImageStream**: `python:3.9-ubi9` (provided out-of-the-box in the `openshift` namespace)
- **Direct Container Image**: `registry.access.redhat.com/ubi9/python-39` (or `ubi8/python-38`)
- **Arbitrary UID compliance**: OpenShift runs containers with an assigned random non-root UID (e.g. `1000650000`) in group `0`. The Red Hat UBI builder automatically sets directory permissions to `g=u` and binds to non-privileged port `8080`.

---

## 3. How to Deploy to OpenShift

### Option A: Declarative Workflow with Binary Source (Fastest & Most Reliable)

Binary builds upload your local directory directly into OpenShift's build pod without requiring code to be pushed to an external Git repository:

```bash
# 1. Login to your OpenShift cluster
oc login https://<api-url>:6443 --token=<your-token>

# 2. Create or switch to a demo project
oc new-project openshift-demos || oc project openshift-demos

# 3. Apply the declarative manifests
oc apply -f k8s/imagestream.yaml
oc apply -f k8s/buildconfig.yaml

# 4. Trigger the S2I build by uploading the local directory
oc start-build s2i-python-demo --from-dir=. --follow

# 5. Now that the image exists, deploy! (Will start in 2 seconds without getting stuck)
oc apply -f k8s/deployment.yaml
oc apply -f k8s/service.yaml
oc apply -f k8s/route.yaml


# 6. Verify the deployment and route
oc rollout status deployment/s2i-python-demo
oc get route s2i-python-demo
```

### Option B: OpenShift CLI `oc new-app` One-Liner

If using a remote Git repository:

```bash
oc new-app python:3.9-ubi9~https://github.com/<your-repo>/openshift-demos.git \
  --context-dir=01-s2i-python-app \
  --name=s2i-python-demo

# Expose route with Edge TLS
oc create route edge s2i-python-demo --service=s2i-python-demo
```

---

## 4. Testing & Verification

### Local Test (Before Deploying)
Run the unit test suite locally:
```bash
python3 test_app.py
```

Run the application locally:
```bash
python3 app.py
# In another terminal:
curl http://localhost:8080/
curl http://localhost:8080/healthz
curl http://localhost:8080/info
```

### In-Cluster Verification
```bash
ROUTE_URL=$(oc get route s2i-python-demo -o jsonpath='{.spec.host}')
curl -k "https://${ROUTE_URL}/"
curl -k "https://${ROUTE_URL}/healthz"
curl -k "https://${ROUTE_URL}/info"
```

---

## 5. S2I Lifecycle Execution Details

When OpenShift executes the build pod for this project:
1. **Source Injection**: OpenShift clones the Git repo or streams the binary tarball into `/tmp/src`.
2. **`assemble` phase**:
   - Reads `.s2i/environment`.
   - Copies `/tmp/src` files to `/opt/app-root/src`.
   - Executes `pip install -r requirements.txt`.
   - Sets group permissions (`chmod -R g+rwX /opt/app-root/src`).
3. **`run` phase**:
   - Starts `gunicorn` listening on `0.0.0.0:8080` executing `app:application`.
