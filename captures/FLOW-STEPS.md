# 5G PDU session release on UPF failure — procedure & capture notes

Captured 2026-10-15 on a live 5G SA core, in reply to the request in open5gs #4860 /
#4863 ("a pcap showing the entire procedure for 5G sessions"). This file describes the
stimulus, the observed message sequence, and where to find each part in the captures.

## Rig

| Item | Value |
|---|---|
| SMF | open5gs `main` @ merge `26f9edd4` ("[SMF] Delete session towards SGW/EPC when last UPF goes down") + follow-up `ae7a88a` |
| UPF | eUPF (edgecomllc), single UPF, native XDP |
| RAN / UE | UERANSIM gNB + 4 UEs × 2 PDU sessions = **8 sessions** (IMSI …082001/.082002 and …83901/.83902) |
| SBI | TLS/mTLS enabled on all SBI legs (so N11/N10/N7 payloads are encrypted in the captures) |
| PFCP / N2 / NAS / N3 | plaintext |
| Addresses | SMF `172.22.0.7`, UPF `172.22.0.8` (host netns), AMF `172.22.0.10`, gNB `192.168.1.168` |
| Clocks | .pcap and open5gs logs are absolute; shown in UTC here (lab local = UTC+8); UERANSIM logs = UTC |

## Stimulus

    docker stop upf          # UPF goes away, no PFCP farewell          (09:22:30)
    ... heartbeat timeout, sessions released, UEs retry for ~36 s ...
    docker start upf         # UPF returns                               (09:23:06)

## Sequence (times UTC, ms precision)

SMF ↔ UPF (PFCP), the core of the scenario:

    09:22:25.794  Heartbeat REQ/RESP — last exchange before the stop
    09:22:30.798  Heartbeat REQ #1 — no response (UPF stopped at 09:22:30.0)
    09:22:33.301  Heartbeat REQ #2        (2.5 s cadence)
    09:22:35.804  Heartbeat REQ #3
    09:22:38.308  give-up: SMF logs "No Heartbeat from UPF" -> "No UPF available";
                  SMF then retries Association Setup Requests towards the dead UPF
                  (~21 retries, ~every 1–2.5 s, until the UPF is back)
    09:23:06.314  Association Setup Request (UPF is back)
    09:23:07.318  Association Setup Response; heartbeats both ways
                  ("Remote PFCP restarted ... in Heartbeat REQ" -> "PFCP restoration" in the SMF log)
    09:23:07.499  Session Establishment Request ×6 (the sessions that re-established)
                  + QER Session Modification requests — all within ~230 ms

Everything else (from NGAP/NAS in the capture and component logs):

    09:22:38.47   UEs receive PDU Session Release Command (5GSM cause = reactivation
                  requested); gNB releases the N2 resources (~09:22:38.47–39.49)
    09:22:38.75+  while the UPF is dead: UE re-establishment attempts are rejected by
                  the SMF ("No UPFs are PFCP associated that are suited to RR"), UEs retry ~1/s
    09:23:07.5+   6/8 sessions re-established automatically after the UPF returned
                  (two earlier runs of the same scenario re-established 8/8 within ~2–4 s)

## Files

- `sn100-bridge-pfcp-ngap-sbi.pcap.gz` — primary capture, 5GC docker bridge, **filtered to**
  PFCP (`udp.port==8805`) + N2 (`sctp`, AMF ↔ gNB, NGAP PPID 0x3c) + SBI (`tcp.port==7777`, TLS).
  Note: an unrelated SGW-C ↔ SGW-U PFCP pair (`172.22.0.5 ↔ 172.22.0.6`) is also present;
  the SMF ↔ UPF traffic is `172.22.0.7 ↔ 172.22.0.8`.
- `ue-eth0.pcap.gz` — UE container interface: NAS over RLS (`udp.port==4997`): release
  command, reactivation, retries, rejects.
- `pfcp-sequence.txt` — decoded PFCP message list for the whole window.
- `FLOW-STEPS.md` — this file.

## Notes

1. SBI is TLS/mTLS in this deployment, so the N11/N10/N7 message contents are not readable
   in the capture; PFCP, NGAP (incl. the NAS PDU inside) and RLS/NAS are plaintext.
2. This run: UEs received 8 release commands; one UE's two sessions did not re-establish
   inside the capture window. Its attempts were answered with
   `No handler for event OGS_EVENT_NAME_SBI_SERVER` (gsm-sm.c:759) while the SMF still held
   stale session objects for it; a clean SMF restart cleared the state. Not root-caused —
   looks like a post-release state-machine edge case, possibly triggered by our test sequence.
3. The capture is filtered to the four stacks above; subscriber-DB and other container
   chatter from the bridge is pruned.
