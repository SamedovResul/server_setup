# MongoDB Data Migration — firstAcademy (Kubernetes)

This documents how to restore/migrate a MongoDB dump into the `firstAcademy` database running inside the `firstacademy-mongo` pod on the cluster, plus troubleshooting steps if things break afterward.

## Prerequisites

- `kubectl` access to the `default` namespace
- `mongorestore` / `mongosh` installed locally (or use the in-pod tools)
- A valid MongoDB dump directory, e.g. `/root/backup/firstAcademy`

## 1. Identify the Mongo pod

```bash
kubectl -n default get pods | grep mongo
```

Note the current pod name — it changes every time the pod restarts (e.g. `firstacademy-mongo-899c8955-vdbf5`).

## 2. Open a port-forward tunnel

```bash
kubectl -n default port-forward pod/<mongo-pod-name> 27018:27017
```

Leave this running in its own terminal. This maps the pod's internal Mongo port (`27017`) to `localhost:27018` on your machine.

## 3. Confirm credentials

Credentials are normally stored in a Kubernetes Secret:

```bash
kubectl -n default get secret mongo-env -o jsonpath='{.data}' | jq
```

Decode the base64 values:

```bash
echo "<base64-username>" | base64 -d
echo "<base64-password>" | base64 -d
```

> **Note:** `MONGO_INITDB_ROOT_USERNAME` / `MONGO_INITDB_ROOT_PASSWORD` env vars only take effect the **first time** a mongod container initializes an *empty* data directory. If the pod restarts and reattaches to an existing PersistentVolumeClaim (PVC), these env vars are ignored — so the Secret can drift out of sync with the actual DB password over time.

## 4. Verify auth actually works

From your host, through the tunnel:

```bash
mongosh --host 127.0.0.1 --port 27018 -u <user> --authenticationDatabase admin
```

If this fails with `AuthenticationFailed`, test directly inside the pod to rule out tunnel/client issues:

```bash
kubectl -n default exec -it <mongo-pod-name> -- mongosh
```

If this connects **without** any `-u`/`-p` (via MongoDB's localhost exception), you're in as an effective admin. From there:

```js
use admin
db.getUsers()
```

This shows every user, which database they're scoped to, and their roles — confirm the user exists under `db: 'admin'` with the expected `roles`.

### Fix a mismatched password

```js
use admin
db.changeUserPassword("<user>", "<new-password>")
```

This takes effect immediately — **no pod restart required.**

### Create the user if it doesn't exist

```js
use admin
db.createUser({
  user: "<user>",
  pwd: "<password>",
  roles: [{ role: "root", db: "admin" }]
})
```

## 5. Run the restore

From your host machine, through the port-forward tunnel:

```bash
mongorestore \
  --host 127.0.0.1 --port 27018 \
  -u <user> --authenticationDatabase admin \
  --db firstAcademy \
  --dir=/root/backup/firstAcademy
```

Omit `-p` to be prompted for the password interactively (avoids it being visible in `ps` output). Watch for command-line wrapping issues — make sure `--dir=/path/to/dump` isn't accidentally split across two lines/arguments, which produces a confusing "provide only one connection string" error.

## 6. Verify the restore

```bash
kubectl -n default exec -it <mongo-pod-name> -- mongosh -u <user> -p <password> --authenticationDatabase admin --eval "use firstAcademy; db.getCollectionNames()"
```

Or from `show dbs` inside `mongosh`, confirm `firstAcademy` now appears with a non-trivial size.

## 7. Post-restore: confirm the app reconnects

Check the backend pod picked up a healthy DB connection:

```bash
kubectl -n default logs <backend-pod-name> --tail=100
```

Look for a clean `Connected to the database` / `Server is listening at port <port>` with no errors.

Test the backend directly, bypassing ingress, to isolate app vs. networking issues:

```bash
kubectl -n default port-forward pod/<backend-pod-name> 4000:4000
curl http://127.0.0.1:4000/api/<some-endpoint>
```

## 8. If the public site 502s but the backend is healthy

This means the break is between ingress and the Service/pod, not the app or DB. Check:

```bash
kubectl -n default get endpoints firstacademy-backend
kubectl -n default get pod <backend-pod-name> -o wide
```

Confirm the IP listed in `endpoints` matches the pod's **current** IP (pods get a new IP on every restart — stale endpoints are a common cause of 502s right after a restart/redeploy).

Also confirm the Service's `targetPort` correctly maps to the app's actual listening port:

```bash
kubectl -n default get svc firstacademy-backend -o yaml
```

## Common gotchas recap

| Symptom | Likely cause |
|---|---|
| `error parsing positional arguments` on mongorestore | `--dir=` path got split across lines/args by copy-paste |
| `AuthenticationFailed` via SCRAM-SHA-1 | Wrong password, wrong `--authenticationDatabase`, or Secret out of sync with actual DB user (stale PVC) |
| Secret's password doesn't match a working login | PVC-persisted data + rotated/changed Secret after initial pod creation |
| `mongosh` connects with no `-u`/`-p` inside the pod | MongoDB localhost exception — use this window to fix/reset the real user |
| Pod deleted, data still present | Data is on a PVC — safe. Confirm with `kubectl get pod ... -o yaml \| grep -A15 volumes` before deleting |
| Backend logs clean, but public site 502s | Ingress → Service → Pod IP mismatch after a pod restart, or Service `targetPort` misconfigured |