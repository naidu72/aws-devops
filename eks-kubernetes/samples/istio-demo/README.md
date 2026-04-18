1. Browser → http://localhost:8080
          ↓
2. Istio Ingress Gateway (port-forward)
   - Matches Gateway resource (demo-gateway)
          ↓
3. VirtualService (frontend)
   - Routes / → frontend service
          ↓
4. Frontend Pod
   - istio-proxy sidecar intercepts request
   - Forwards to nginx container
   - nginx serves HTML page
          ↓
5. User clicks "Call Backend API"
   - JavaScript: fetch('/api')
          ↓
6. nginx proxy_pass → http://backend:8080
   - istio-proxy sidecar intercepts
          ↓
7. VirtualService (backend)
   - 90% to backend subset v1
   - 10% to backend subset v2
          ↓
8. DestinationRule (backend)
   - Selects pods based on version label
          ↓
9. Backend Pod (v1 or v2)
   - istio-proxy sidecar intercepts
   - Forwards to backend container
   - Returns JSON response
          ↓
10. Response flows back through sidecars
    - Full telemetry collected
    - mTLS encryption between services
