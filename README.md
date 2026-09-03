# Titan Call Frontend

Production dashboard for Titan Call on GetUrClarity.net.

## Runtime
- Svelte 5 + Vite
- Motion for page transitions
- Live data only from Titan Call control API
- Default API: `https://api.geturclarity.net`
- Override with `VITE_TITAN_CALL_API`

## Production domain
`call.geturclarity.net`

The UI intentionally shows disconnected/empty states when the backend is unavailable. It does not generate mock telemetry.
