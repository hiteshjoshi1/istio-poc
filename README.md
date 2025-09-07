# Istio Poc

The idea is to play with Istio and setup

- mTLS (done)
- Rate limiting at the gateway 
- WAF (optional, via WASM/Coraza) at the gateway 
- Zero‑code telemetry (metrics, logs, traces)
- AuthN: verify user JWT (issuer, signature, audience)
- AuthZ: identity‑based and claims‑based policies
- Allow only callers with payments:write scope or role=admin

Actual ReadMe: 

https://www.notion.so/Istio-Example-266c54edb687802d89bbea9096c75ee3#266c54edb68780c0ad7ccb74a3873696