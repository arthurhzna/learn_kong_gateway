# Kong Cheat Sheet: Route Path, Strip Path, Service Path

> Study notes: how Kong **cuts** and **joins** the path before a request reaches the backend.

---

## 1. The three fields that decide the path

| Field | Where in the dashboard | What it does | Symbol |
|---|---|---|---|
| **Route → Paths** | Route form | **Matches** the request to a backend (always) + **cuts** it (only when Strip is YES) | 🔍 + ✂️ |
| **Route → Strip Path** | Route form | Switch: cut the Route Path or not | 🔘 YES / NO |
| **Service → Path** | Service form | **Joins** (prepends) text in front of the path | ➕ |

```
Route Path   = 🔍 matcher  +  ✂️ cutter (when Strip is YES)
Strip Path   = 🔘 cutter on/off switch
Service Path = ➕ prefix that gets joined in front
```

---

## 2. The formula

```
Backend receives = Service Path  +  ( client URL  −  Route Path , if Strip is YES )
                   └── join ───┘     └──────────────── cut ──────────────────────┘
```

---

## 3. How a request flows through Kong

```
                 Path from the client
   /micro/hb-service/heartbeat/v2/insert_data
                       │
                       ▼
   ┌──────────────────────────────────────────────┐
   │ ① MATCH (Route Path)                          │
   │    "starts with /micro/hb-service?"  ✅        │
   │    → use Service hb-service (rudeus)          │
   └──────────────────────┬───────────────────────┘
                          ▼
   ┌──────────────────────────────────────────────┐
   │ ② CUT? (Strip Path)                           │
   │    YES → remove the Route Path part  ✂️        │
   │    NO  → keep the path as is                  │
   └──────────────────────┬───────────────────────┘
                          ▼
   ┌──────────────────────────────────────────────┐
   │ ③ JOIN (Service Path)                         │
   │    prepend the Service Path  ➕                │
   │    (empty → nothing is added)                 │
   └──────────────────────┬───────────────────────┘
                          ▼
        http://<Service Host>:<Port>  + resulting path
```

---

## 4. Strip YES vs Strip NO

Route Path `/micro/hb-service`, Service Path **empty**:

```
Strip NO  ─────────────────────────────────────────────────────────
  in    : /micro/hb-service/heartbeat/v2/insert_data
  cut   : (nothing)
  out   : /micro/hb-service/heartbeat/v2/insert_data   ← unchanged

Strip YES ─────────────────────────────────────────────────────────
  in    : /micro/hb-service/heartbeat/v2/insert_data
          ✂️────────────────────┘
  out   : /heartbeat/v2/insert_data                        ← cut
```

| | Strip **YES** | Strip **NO** |
|---|---|---|
| Route Path used for matching? | ✅ | ✅ |
| Route Path cut off? | ✅ | ❌ |
| Route Path required? | ✅ | ✅ (still needed to pick the backend) |

> ⚠️ Strip NO **only turns off the cutting**. The Route Path is **still required**, because it tells Kong which backend to use. If it is empty, the Route becomes a *catch-all* and can clash with other Routes.

---

## 5. Cut and join, shown as text pieces

```
Example B: Service Path = /micro/hb-service, Route Path = /hb, Strip YES

  in      [ /hb ][ /heartbeat/v2/insert_data ]
             ✂️ removed
          [ /heartbeat/v2/insert_data ]
             ➕ joined in front
  out     [ /micro/hb-service ][ /heartbeat/v2/insert_data ]
```

In effect, Kong **swaps** `/hb` for `/micro/hb-service`.

---

## 6. Examples

**Backend (Go code):** `/micro/hb-service/heartbeat/v2/insert_data`
**Service Host:Port:** `rudeus-svc:80`
**`KONG`** = `https://kong-gateway-uat.example.com`

### ✅ Correct

| # | Service Path | Route Path | Strip | Client calls | Backend receives |
|---|---|---|---|---|---|
| A ⭐ | *(empty)* | `/micro/hb-service` | **NO** | `KONG/micro/hb-service/heartbeat/v2/insert_data` | `/micro/hb-service/heartbeat/v2/insert_data` |
| B | `/micro/hb-service` | `/hb` | **YES** | `KONG/hb/heartbeat/v2/insert_data` | `/micro/hb-service/heartbeat/v2/insert_data` |
| C | `/micro/hb-service/heartbeat` | `/heartbeat-api` | **YES** | `KONG/heartbeat-api/v2/insert_data` | `/micro/hb-service/heartbeat/v2/insert_data` |
| D | `/micro/hb-service/heartbeat/v2/insert_data` | `/insert-hb` | **YES** | `KONG/insert-hb` | `/micro/hb-service/heartbeat/v2/insert_data` (one endpoint only) |

⭐ = the setup the team uses (`/micro/<name>` pattern + Strip NO)

### ❌ Wrong (common mistakes)

| # | Service Path | Route Path | Strip | Backend receives | Problem |
|---|---|---|---|---|---|
| W1 | *(empty)* | `/micro/hb-service` | **YES** | `/heartbeat/v2/insert_data` | Prefix lost → 404 |
| W2 | `/micro/hb-service` | `/micro/hb-service` | **NO** | `/micro/hb-service/micro/hb-service/...` | Doubled → 404 |
| W3 | `/micro/hb-service` | `/hb` | **NO** | `/micro/hb-service/hb/heartbeat/...` | Stray `/hb` → 404 |
| W4 | `/micro/hb-service` | `/hb/` ← trailing `/` | **YES** | `/micro/hb-serviceheartbeat/...` | Glued together → 404 |

> W4 happens because **Path handling = v1**: Kong joins the pieces as they are, without adding a `/`.
> Rule: **don't end a Route Path with `/`**.

### A backend whose code has NO prefix

Code: `router.Handle("/users/list")`, Route Path `/micro/roxy`, Service Path empty

| Strip | Client calls | Backend receives | Result |
|---|---|---|---|
| **YES** | `KONG/micro/roxy/users/list` | `/users/list` | ✅ |
| **NO** | `KONG/micro/roxy/users/list` | `/micro/roxy/users/list` | ❌ |

### Empty Route Path (catch-all, e.g. `pvj-ucok`)

Service Path `/pvj-ucok/api/v4`, Route Path **empty**, Methods GET/POST

```
Client : KONG/user/profile
Cut    : (nothing to cut → Strip has no effect)
Join   : /pvj-ucok/api/v4 + /user/profile
Backend: http://pvj-svc:80/pvj-ucok/api/v4/user/profile
```

> A Route with no path acts as a **fallback**: every GET/POST that doesn't match another Route lands here.

---

## 7. Choosing the right setup

Compare **the path the client sends** with **the path in the backend code**:

```mermaid
flowchart TD
    A[Client path vs backend code path] --> B{Exactly the same?}
    B -- Yes --> C[Service Path empty<br/>Strip NO<br/>Route Path = prefix in code]
    B -- No --> D{Client has a prefix<br/>that is NOT in the code?}
    D -- Yes --> E[Strip YES<br/>Route Path = client prefix]
    D -- No --> F{Code has a prefix<br/>the client does NOT send?}
    F -- Yes --> G[Fill Service Path<br/>with that prefix]
    E --> H{After cutting,<br/>still missing a prefix?}
    H -- Yes --> G
    H -- No --> I[Service Path empty]
```

Text version:

| Situation | Setup |
|---|---|
| Client path **is the same** as the code | Service Path empty + **Strip NO** |
| Client has an **extra** prefix that isn't in the code | **Strip YES** |
| Code has a prefix the client **doesn't send** | Put that prefix in **Service Path** |

---

## 8. Route Path rules

1. **Specific to one backend**: `/micro/hb-service`, not `/micro`.
2. **No trailing `/`**: `/hb`, not `/hb/`.
3. For **Strip NO + empty Service Path**: the Route Path **must match** the start of the path in the backend code.
4. Kong picks the Route with the **longest matching prefix** (the most specific one).

```
Request: /micro/roxy/users
  Route /micro          ✅ matches
  Route /micro/roxy  ✅ matches, longer → CHOSEN

Request: /micro/eris/x   (eris has no Route yet)
  Route /micro          ✅ matches → goes to the /micro backend by mistake  ⚠️
  (without a /micro Route → Kong returns 404 "no Route matched", which is clearer)
```

---

## 9. The big picture: from client to container

```
Client / Postman
  │  https://kong-gateway-uat.example.com/micro/hb-service/heartbeat/v2/insert_data
  ▼
OpenShift Route  (kong-data-plane)      ← entry into the cluster (≈ "ports:" in docker-compose)
  ▼
Kong Data Plane
  │  ① match Route Path  ② cut (Strip)  ③ join (Service Path)
  ▼
K8s Service  rudeus-svc:80                   ← stable address (DNS .svc.cluster.local)
  ▼
Endpoints → Pod rudeus  :9036                    ← backend container
  ▼
router.Handle("/micro/hb-service/heartbeat/v2/insert_data")  ✅
```

Direct access **without Kong** (no plugins, path not changed):

```
https://heartbeat-uat.example.com/micro/hb-service/heartbeat/v2/insert_data
```

---

## 10. Quick checklist before creating a Service + Route

- [ ] Check the path in the backend code (`router.Handle(...)`)
- [ ] Decide the URL the client will call
- [ ] Service → Host = K8s Service name (e.g. `rudeus-svc`), Port = `80`
- [ ] Service → Path = the **missing** part (empty if nothing is missing)
- [ ] Route → Paths = a **specific** prefix, no trailing `/`
- [ ] Route → Strip = **YES** if the client prefix isn't in the code, **NO** if it is
- [ ] Check: `Service Path + (client − Route Path if YES)` = path in the code? ✅


# Kong: Route Hosts

> What the **Hosts** field on a Kong Route does (Konga / Kong Manager / `kong.yml`).
> Domain and service names below are **placeholders**.

| Placeholder | Meaning |
|---|---|
| `kong-proxy.apps.example.com` | Kong proxy domain (OpenShift Route to the Kong data plane) |
| `heartbeat.apps.example.com` | OpenShift Route that goes straight to the backend (no Kong) |
| `heartbeat-svc` | Kubernetes Service of the backend |
| `/micro/hb-gateway` | Path prefix in the backend code |

---

## 1. A URL has two parts

```
https://kong-proxy.apps.example.com/micro/hb-gateway/heartbeat/v2/insert
        └─────────── HOST ─────────┘└──────────────── PATH ──────────────┘
          checked by Route Hosts         checked by Route Paths
```

A Kong Route can check **both**:

| Route field | Checks | Changes the request? |
|---|---|---|
| **Hosts** | Domain the client called (`Host` header) | ❌ No, match only |
| **Paths** | Path the client called | Only if Strip Path is YES (cut) |
| **Methods** | GET / POST / … | ❌ No |

---

## 2. Analogy: a guard at the door

Kong is a **building**, and every Route is a **room** with a guard at the door.

```
                ┌──────────────── Route heartbeat_v2 ────────────────┐
Request ──►  🧍 Guard:                                               │
             1. "Which address (Host) did you come through?"         │
                → must be kong-proxy...  OR  heartbeat...            │
             2. "Where are you going (Path)?"                        │
                → must start with /micro/hb-gateway                  │
                                                                     │
             Both correct ──► enter ──► heartbeat-svc                │
                └─────────────────────────────────────────────────────┘
```

---

## 3. Matching rules: OR inside, AND between

- **Inside one field: OR**. Any one listed value is enough.
- **Between fields: AND**. Every filled field must match.

```
Hosts  : kong-proxy.apps.example.com   OR   heartbeat.apps.example.com
   AND
Paths  : starts with /micro/hb-gateway
   AND
Methods: (if set)
```

| Request | Host | Path | Result |
|---|---|---|---|
| `https://kong-proxy.apps.example.com/micro/hb-gateway/heartbeat/v2/insert` | ✅ | ✅ | ✅ matches this Route |
| `https://kong-proxy.apps.example.com/user/profile` | ✅ | ❌ | ❌ try other Routes |
| `https://kong-proxy-uat.apps.example.com/micro/hb-gateway/...` | ❌ | ✅ | ❌ 404 `no Route matched` |

---

## 4. Hosts filled vs empty

| Request domain | Hosts **filled** | Hosts **empty** |
|---|---|---|
| In the list | ✅ accepted | ✅ accepted |
| **Not** in the list | ❌ rejected (404) | ✅ accepted |

```
Hosts empty   = "any domain is fine, only the Path is checked"
Hosts filled  = "only domains in the list are accepted"
```

The difference **only** shows for domains that are **not** in the list.

---

## 5. Why does one Route have two Hosts?

Two hosts = **two doors** into the same room.

```
https://kong-proxy.apps.example.com/micro/hb-gateway/...  ─┐
                                                            ├─► Route heartbeat_v2 ─► heartbeat-svc
https://heartbeat.apps.example.com/micro/hb-gateway/...   ─┘
```

**But** Hosts only matter for requests that **have already reached Kong**.

```
Layer 1: OpenShift Route   (decides whether the request reaches Kong at all)
Layer 2: Kong Route Hosts  (only works once the request is inside Kong)
```

| Domain | OpenShift Route points to | Host on the Kong Route used? |
|---|---|---|
| `kong-proxy.apps.example.com` | Kong data plane Service | ✅ Yes |
| `heartbeat.apps.example.com` | **Directly** to `heartbeat-svc` | ❌ No, Kong never sees this traffic |

The second domain is usually listed as:

1. **Migration prep.** Later the OpenShift Route `heartbeat` is switched to Kong, and apps using the old URL keep working without changing it.
2. **Leftover config.**

Check whether a domain goes through Kong (Kong adds `Via` / `X-Kong-*` headers). This is read-only:

```bash
curl -sk -o /dev/null -D - https://heartbeat.apps.example.com/ | grep -iE "via|x-kong"
```

---

## 6. Don't mix them up: Route Hosts vs Service Host

```
Client ──(Route Hosts)──► KONG ──(Service Host)──► Backend
         INCOMING domain           OUTGOING target
```

| | Route → **Hosts** | Service → **Host** |
|---|---|---|
| Direction | Client → Kong (**incoming**) | Kong → backend (**outgoing**) |
| Value | Domain the client called | Kubernetes Service name of the backend |
| Example | `kong-proxy.apps.example.com` | `heartbeat-svc` |
| How many | A list | One |
| Purpose | Matching | Destination |

---

## 7. Hosts and Route priority

A Route with **more conditions** (Hosts + Paths + Methods) is **more specific**, so Kong prefers it.

| Route | Conditions | Priority |
|---|---|---|
| `heartbeat_v2` | Hosts + Paths | High |
| Route with Paths only | Paths | Medium |
| Catch-all (Methods only) | Methods | Low (fallback) |

That's why heartbeat requests don't fall into a catch-all Route.

---

## 8. Things to remember

1. **Copying config between environments (dev → uat):** change Hosts to the target environment's domains. If UAT Kong still lists dev domains, every UAT request returns **404 `no Route matched`**.
2. **Emptying Hosts:** the Route accepts every domain. Fine when there's one Route per Path, but it can **clash** with another Route that uses the same Path and is told apart only by Hosts.
3. **Wildcards:** Kong supports `*.example.com` (leading or trailing wildcard) to match many subdomains at once.
4. **`preserve_host`** (another Route field): default `false`, so Kong sends `Host: heartbeat-svc` (the Service Host) to the backend. When `true`, Kong forwards the client's original domain.

---

## 9. Try it on a local Kong

`kong.yml`:

```yaml
_format_version: "3.0"
services:
- name: echo
  url: http://httpbin.org/anything
  routes:
  - name: echo
    paths: [/echo]
    hosts: [domain-a.test]
```

You can fake the `Host` header with `-H`, so no real domain is needed:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8000/echo -H "Host: domain-a.test"
```

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8000/echo -H "Host: domain-b.test"
```

| Setting | `domain-a.test` | `domain-b.test` |
|---|---|---|
| `hosts: [domain-a.test]` | 200 | 404 |
| `hosts` removed | 200 | 200 |

---

## 10. One-line summary

> **Route Hosts = "the request must come through this domain".** Match only, changes nothing, and only applies to requests that have already reached Kong.
