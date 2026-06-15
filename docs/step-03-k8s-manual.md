# Step 03 — Deploy to Kubernetes (manual manifests)

**Goal:** deploy your image to Kubernetes by hand. Externalize configuration into a **ConfigMap** and the database URI into a **Secret**. MongoDB stays **outside** the cluster for now (the app connects to a Mongo on your host).

You write every manifest yourself from the requirements and hints below. There is **no copy-paste YAML here** — figure out the fields. (If you get stuck, the reference answer is in `solved/step-03/`.)

> **Set up:** make a `k8s/` folder at the repo root for the manifests below — you'll keep evolving this same folder through the later steps. You also need the image from Step 02 pushed to Docker Hub (`<dockerhub-user>/movie-api:1.0`).

---

## A. ConfigMap — non-secret config

**Goal:** hold the app's non-sensitive setting `PORT=3000`.

**Task:** write `k8s/configmap.yaml` defining a `ConfigMap` named `movie-api-config` whose `data` has a `PORT` key set to `"3000"`.

*Hints:*
- `apiVersion: v1`, `kind: ConfigMap`.
- Values under `data:` must be **strings** — quote the number.

## B. Secret — the database URI

**Goal:** keep `MONGO_URI` out of the ConfigMap because it can carry credentials.

**Task:** write `k8s/secret.yaml` defining an `Opaque` Secret named `movie-api-secret` that holds `MONGO_URI`. For now point it at a Mongo reachable from the cluster on your host.

*Hints:*
- `kind: Secret`, `type: Opaque`.
- Use `stringData:` (not `data:`) so you can write the value in plain text — Kubernetes base64-encodes it for you.
- From inside minikube / Docker Desktop, your host is reachable as `host.docker.internal`. So the value looks like `mongodb://host.docker.internal:27017/movies`.

> **The Mongo container must publish its port to the host.** The pods run *inside* the cluster and reach your host through `host.docker.internal`, so Mongo has to be listening on the host network — not just on Docker's internal bridge. Run the `mongo:7` container from Step 01 with the host port published (`-p 27017:27017`); without it the container runs but the cluster can't connect, and the app pods fail their readiness probe.
>
> _Hint:_ confirm the port is published before deploying — `docker ps` should show the Mongo container mapping `0.0.0.0:27017->27017/tcp`.
>
> _On minikube/Linux:_ if `host.docker.internal` doesn't resolve, point `MONGO_URI` at your host's LAN IP instead and open port `27017` in the firewall.

## C. Deployment — the app

**Goal:** run 2 replicas of your image, inject the ConfigMap + Secret as env vars, and add health probes.

**Task:** write `k8s/deployment.yaml` for a `Deployment` named `movie-api`.

*Requirements & hints:*
- `apiVersion: apps/v1`, `kind: Deployment`, `replicas: 2`.
- Label the pods (e.g. `app: movie-api`) and make sure the `selector.matchLabels` matches the pod template labels — a mismatch is the #1 reason a Deployment won't come up.
- Container image: `<dockerhub-user>/movie-api:1.0`, container port `3000`.
- Inject **all** keys from the ConfigMap and Secret at once — look up `envFrom` with `configMapRef` and `secretRef` (simpler than listing each `env` var).
- Add a `livenessProbe` and a `readinessProbe`, both an `httpGet` on path `/health`, port `3000`.

*Self-check:* what happens to the pod if `/health` starts failing? (liveness vs. readiness)

## D. Service — stable access

**Goal:** give the Deployment a stable in-cluster address.

**Task:** write `k8s/service.yaml` for a `Service` named `movie-api`.

*Hints:*
- `type: ClusterIP`.
- `selector` must match the pod labels from the Deployment.
- Expose `port: 80` and forward to the container's `targetPort: 3000`.

---

## E. Apply and verify

**Tasks (work out the commands):**
1. Apply the whole `k8s/` folder at once.
2. List the app pods and confirm they reach `Running`/ready.
3. Tail the Deployment's logs and confirm it printed that it connected to your host Mongo.
4. Reach the Service from your machine (see the two options below) and curl `/health`, then create and list a movie.

*Hints:*
- `kubectl apply` takes `-f <file-or-dir>`.
- `kubectl get pods -l app=movie-api`, `kubectl logs deploy/movie-api`.

Now reach the service from your machine and hit `/health`. You have two options:

- **Option A — `minikube service` (preferred on minikube).** Ask minikube for the service URL (`minikube service movie-api --url`), then `curl` the URL it prints. Because the cluster runs inside a VM/container, this is the reliable way to reach a `ClusterIP`/`NodePort` Service.
- **Option B — `kubectl port-forward` (any cluster).** Forward a local port to the Service and `curl localhost`. Works everywhere but ties up the terminal while the tunnel stays open.

> **`ImagePullBackOff` / `ErrImagePull` — or skipping the registry on purpose (minikube/kind).** If the pod can't pull the image — an anonymous Docker Hub rate limit (`pull access denied` / `toomanyrequests`), or you simply never pushed to a registry — sidestep the pull entirely: pull (or build) the image on your host, then load it into the cluster's container runtime with `minikube image load <image-tag>`. Confirm it landed with `minikube image ls`.
>
> _Hint:_ point the container `image:` at the tag you loaded and set an `imagePullPolicy` that won't force a registry pull — `IfNotPresent` (use the loaded image if present) or `Never` (fail rather than pull). Then re-apply the manifests and `kubectl rollout restart deploy/movie-api` to pick up the change.

---

## What you learned

- Config and secrets are externalized from the image: the same image runs anywhere, configured by a ConfigMap + Secret via `envFrom`.
- A Service gives the Deployment a stable name and port inside the cluster.

## Next

→ [Step 04 — MongoDB on Kubernetes](step-04-k8s-mongodb.md)
