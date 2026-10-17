# kingkongg

Catatan belajar Kong Gateway — cara menjadi kingkong 🦍

> Semua nama service, IP, dan port asli sudah disensor. Ini catatan pribadi doang.

## Service ngacu ke mana?

Cek service:

```bash
oc get svc <nama-service>
```

```
NAME             TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
<nama-service>   ClusterIP   x.x.x.x      <none>        80/TCP    2y276d
```

Port `80` di Service itu cuma pintu depan — semua trafik diteruskan ke port container (`<port-container>`).

Cek endpoints (daftar pod di belakang Service):

```bash
oc get endpoints <nama-service>
```

```
NAME             ENDPOINTS   AGE
<nama-service>   <none>      2y276d
```

## Alur trafik

```
Kong / pod lain
      │  http://<nama-service>:80
      ▼
┌──────────────────────────────────┐
│ Service  <nama-service>          │  ← alamat tetap (IP + DNS + port 80)
│ x.x.x.x:80                       │
└────────────────┬─────────────────┘
                 │ "ke mana diteruskan?" → lihat daftar:
                 ▼
┌──────────────────────────────────┐
│ Endpoints <nama-service>         │  ← daftar IP pod yang Ready
│   sekarang:  <none>              │
│   nanti:     y.y.y.1:<port>      │
│              y.y.y.2:<port>      │  (kalau 2 replika → dibagi bergantian)
└────────────────┬─────────────────┘
                 ▼
          Pod :<port-container>
```

## Catatan

- Endpoints `<none>` karena DevOps lagi matiin semua container.
- Endpoints **diisi otomatis oleh Kubernetes** — begitu ada pod yang Ready dan cocok dengan selector Service, IP-nya langsung masuk ke daftar.
