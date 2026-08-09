---
name: lan-front-door
description: Giving self-hosted services real names and trusted HTTPS on a home network — one front-door reverse proxy, an internal certificate authority, and name resolution when the router refuses to help. Use when a LAN service is still reached by ip:port, when a browser forces HTTPS onto a plain-HTTP homelab page, when edits to a mounted config file do not appear inside the container, or when deciding what belongs on the always-on server versus a cluster. Complements the `lan-app` skill, which covers building one app; this covers the layer in front of all of them.
---

# A front door for the whole network

The end state: every internal service answers on a name, over HTTPS, with no
port number and no certificate warning, and no traffic leaves the house.

```
        hosts file / LAN DNS
                 │
                 ▼
        anchor server :443          one Caddy, host networking
        internal CA issues certs
                 │
   ┌─────────────┼─────────────┐
   ▼             ▼             ▼
service :6157  app :8100   dashboard :8090
```

Two layers of reverse proxy is correct here, not redundant. Each app keeps its
own Caddy inside its compose project — that one knows about the app's `web` and
`api` containers and its SSE flush settings. The front door knows only about
names and host ports. Neither needs to learn the other's job.

The front door runs with host networking so it can reach every published port
without being joined to each compose network.

## Use `home.arpa`

RFC 8375 reserves `home.arpa` for exactly this. It will never be delegated, so
it cannot collide with a real domain the way `.local` (mDNS), `.home`, `.lan`
or a domain someone else owns eventually can.

## The failures that have already happened

**Bind-mounting a single config file makes your edits invisible.** Mount the
*directory*:

```bash
-v "$HOME/caddy:/etc/caddy:ro"      # not .../Caddyfile:/etc/caddy/Caddyfile:ro
```

A single-file bind mount binds an **inode**. Vim — like most editors, and like
`sed -i` — does not write through the existing inode; it writes a new file and
renames it over the old name. The directory entry now points somewhere new and
the container is still holding the original inode. The file on the host is
correct, the file in the container is stale, a reload "succeeds", and nothing
anywhere reports a problem. This is the same class of error as trusting a build
you did not verify: check the artifact that is actually running.

```bash
docker exec <proxy> cat /etc/caddy/Caddyfile | head    # what the proxy really has
```

**A consumer router may refuse to let you set DNS.** A LAN-wide resolver is the
right answer — one place to add a name, every device gets it. But some
router subscriptions disable custom DNS whenever their own filtering and
security features are on, and the choice is one or the other. Losing the
router's filtering to gain convenient names is usually the worse trade, so the
fallback is a hosts file per device. It works everywhere, needs no
infrastructure, and costs one edit per device per new name. Write the bootstrap
script the first time, not the third.

**`sed -i` is not portable.** GNU `sed` takes `-i`; the BSD `sed` on macOS
requires an argument to it, so the same script silently misbehaves or fails on
the other half of a mixed fleet. For a script that must run on both, use `awk`
into a temp file and move it into place:

```bash
awk -v ip="$IP" '
  BEGIN { split(names, a, " ") }
  { keep = 1; for (i in a) if ($2 == a[i]) keep = 0; if (keep) print }
  END { for (i in a) print ip "\t" a[i] }
' names="$NAMES" /etc/hosts > "$tmp" && sudo mv "$tmp" /etc/hosts
```

Make it idempotent — strip any existing line for each name before appending —
so re-running after an IP change corrects rather than duplicates.

**Browsers now assume HTTPS, and you will not win that argument.** Chrome's
https-first behaviour, HSTS preloading and typed-name upgrading all push a bare
hostname to `https://`. Declaring `http://name.home.arpa` sites in the proxy
means fighting the browser forever. Serve real TLS instead:

```caddy
{
    local_certs
}

service.home.arpa {
    reverse_proxy <anchor-ip>:6157
}
```

`local_certs` makes the proxy run its own CA and issue certificates from it.
Export the root and trust it on each device:

```bash
docker cp <proxy>:/data/caddy/pki/authorities/local/root.crt ./root.crt
```

macOS accepts it through Keychain Access readily. On Windows, `certutil` may
fail depending on shell and privilege level — importing into **Trusted Root
Certification Authorities** through the certificate manager UI works. Trust the
root, not each leaf, or every new service costs another install.

Understand what this buys and what it does not. It encrypts LAN traffic and
removes warnings. It is not authentication: anything on the network still
reaches these services. Say so in the README rather than letting HTTPS imply a
security posture the setup does not have.

**Publishing on `0.0.0.0` exposes services to every interface, including your
overlay network.** `-p 6157:6157` binds all interfaces; if the host also runs a
VPN or mesh interface, the service is reachable from that too. Bind explicitly:

```bash
-p <lan-ip>:6157:6157
```

Audit this per container rather than per intention — the ones that matter most
are the management surfaces, since a container that can control the container
runtime is worth more to an attacker than any app behind it.

**A container run in the foreground does not survive a reboot.** Run detached
with `--restart unless-stopped`, then actually reboot something once, or accept
that you have not tested it.

**`Running` and `1/1` describe the process, not the purpose.** A management
agent sat healthy for ninety minutes doing nothing but logging an error every
eight seconds, because the generated manifest that deployed it had a
placeholder left unsubstituted:

```yaml
data:
  EDGE_ID: ""      # the console generates a UUID here; applying the template
                   # verbatim leaves the field present, empty, and valid YAML
```

Every liveness signal was green — the process was up, the port was listening,
the readiness probe passed — because none of them know what the container is
*for*. An empty string is not a missing key, so nothing rejected it. Read the
logs of the thing that claims to be fine, especially when something downstream
of it is unexplainably broken; a component reporting its own health is
reporting on itself, not on the job.

The generalisation worth keeping: when a vendor's UI generates a manifest, the
UI is substituting values into it. Applying the raw template from the docs
instead skips the substitution, and the omission is usually silent.

## Verifying without touching a hosts file

`curl --resolve` maps a name to an address for one request, so the proxy's
virtual-host routing and certificate can be tested from a machine that has no
hosts entry and no trusted root:

```bash
curl -skS -o /dev/null -w '%{http_code}\n' \
  --resolve name.home.arpa:443:<anchor-ip> https://name.home.arpa/
```

Compare the response through the front door against the response straight to
the backend port. If both return the same thing, the proxy is not your problem
— which is worth establishing before changing proxy config to fix something
that turns out to be a browser cache or an untrusted root.

For anything with a login, check that a state-changing request survives the
proxy too. Some applications validate the `Origin` header against the host and
reject proxied writes while serving reads perfectly, so a page that loads is
not evidence that the app works. One unauthenticated `POST` with a deliberately
empty body distinguishes the cases: a complaint about the *payload* means the
request reached the handler, while a complaint about a token or origin means
the proxy is being rejected.

## Anchor and cluster

When a cluster arrives, the question is whether the existing always-on server
joins it. Keeping it out is the defensible default:

```
anchor server           cluster nodes
  storage                 clustered compute
  stateful services       scheduled workloads
  media                   things that may be rescheduled
  the front door
```

The anchor holds what must not move: disks, databases, the media library, and
the proxy every name points at. The cluster holds what can be rescheduled by
definition. Folding the anchor in would make the machine holding all the state
also the machine whose workloads a scheduler is entitled to move.

The cost is honest: two systems to operate, two update paths, and a front door
that must eventually route to cluster-side services as well as local ports.

**Reaching a cluster service from outside: the front door is usually enough.**
A NodePort answers on every node, but a management tool is configured with one
address, so naming a node pins the service's availability to that node being
powered on. The textbook fix is a virtual IP — kube-vip or keepalived — and
that *is* the right answer for the API server on `:6443`, where clients and
joining nodes need one endpoint that survives a control-plane node going away.

For everything else, check where the client lives first. If the client already
runs on the proxy host, routing the hop through the front door adds no failure
domain the client does not already have, and it costs a config block instead of
a daemon on every node and an address reserved outside the DHCP pool:

```caddy
https://<host-ip>:30778 {
    reverse_proxy <node1>:30778 <node2>:30778 {
        lb_policy first          # every node forwards to the same pod;
        lb_try_duration 10s      # spreading connections buys nothing
        health_uri /ping
        health_interval 10s
        transport http { tls tls_insecure_skip_verify }
    }
}
```

Configure the client with the **IP**, not a name. A containerised client
resolves through the container runtime's DNS, not the host's `/etc/hosts`, so a
hosts-file name does not exist as far as it is concerned. Add the name for
humans, and know which of the two the machine is using.

**Test the failover, do not just configure it.** Put a *blackholed* address
first in the pool — one that drops packets, which is what a powered-off machine
looks like, rather than one that refuses the connection, which is what a
running machine with a stopped service looks like. Those are different failures
and only one of them resembles the case you care about. Reload, wait out one
health-check interval, and time a real request. Failover here measured 37 ms,
because the active check had already evicted the dead upstream before the
request arrived — a number worth knowing rather than assuming.

**Find the nodes by probing, not from an inventory.** Assumed-consecutive
addresses were wrong within a week of setup here — DHCP had put the nodes
elsewhere. A sweep for the ports a node actually serves (`22`, the API port,
the kubelet port) locates them in seconds and tells you how many are really up,
which is the number that matters and is frequently not the number in the plan.
An unauthenticated request to the API port returns `401`; that is a *success*
for discovery purposes, because only a real API server answers that way. Read
the certificate's SAN list to confirm which nodes belong to the same cluster.

## Say what is not done

The pattern in these repos: finish the sentence "this works, except…". Trusted
HTTPS on a name says nothing about whether the certificate is trusted on the
phone, whether the management endpoint is bound too widely, or whether the
reboot has been tested. Each of those is one command to check and a permanent
false belief if skipped.
