# Demo 2: Node.js Source-to-Image (S2I) on OpenShift

This demo demonstrates building and running a Node.js web application on OpenShift using Red Hat's official UBI 9 Node.js S2I builder (`nodejs:18-ubi9` or `nodejs:20-ubi9`).

---

## 1. What This Demo Contains

- **`server.js`**: Production-ready HTTP service with native JSON endpoints, graceful shutdown handler (`SIGTERM`/`SIGINT`), and arbitrary UID support.
- **`package.json`**: Standard NPM scripts (`npm start`, `npm test`) and engines declaration.
- **`.s2i/environment`**: Declares `NODE_ENV=production`, `PORT=8080`, and `NPM_RUN=start`.
- **`k8s/`**: Manifests for ImageStream, BuildConfig (Source strategy), Deployment, Service, and Edge-terminated Route.
- **`test_server.js`**: Local unit testing suite verifying all HTTP status codes and responses.

---

## 2. Red Hat Builder Details

- **Official Base Image**: `registry.access.redhat.com/ubi9/nodejs-18` (or `nodejs-20`)
- **Default ImageStream**: `nodejs:18-ubi9` (in the `openshift` namespace)
- **Assembly behavior**:
  - Automatically runs `npm install --omit=dev` if a `package.json` is detected.
  - Honors `NODE_ENV=production` for optimized build and runtime memory footprint.
  - Runs as arbitrary assigned non-root user (e.g. UID `1000650000`).

---

## 3. Quickstart Deployment Guide

### Deploy via OpenShift CLI (Binary Build)

```bash
# 1. Switch or create project
oc new-project openshift-demos || oc project openshift-demos

# 3. Apply the declarative manifests
oc apply -f k8s/imagestream.yaml
oc apply -f k8s/buildconfig.yaml

# 4. Trigger the S2I build by uploading the local directory
oc start-build s2i-nodejs-demo --from-dir=. --follow

# 5. Now that the image exists, deploy! (Will start in 2 seconds without getting stuck)
oc apply -f k8s/deployment.yaml
oc apply -f k8s/service.yaml
oc apply -f k8s/route.yaml

# 6. Watch rollout
oc rollout status deployment/s2i-nodejs-demo

# 7. Retrieve route hostname
ROUTE=$(oc get route s2i-nodejs-demo -o jsonpath='{.spec.host}')
curl -k "https://${ROUTE}/"
curl -k "https://${ROUTE}/healthz"
```

---

## 4. Local Testing

```bash
# Run unit tests
node test_server.js

# Or start the server locally
node server.js
# In a second shell:
curl http://localhost:8080/info
```
