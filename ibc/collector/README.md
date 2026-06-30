# IBC Relayer Activity Collector

Script Node.js que recoge el historial de txs IBC de cada wallet relayer de Cumulo y genera un JSON acumulativo consultado por el dashboard.

---

## Ubicación en servidor

| Recurso | Ruta |
|---|---|
| Script | `/opt/ibc-relayer-collector/collector-relayers.js` |
| JSON generado | `/var/lib/cumulo-ibc-relayer/activity.json` |
| Servicio systemd | `ibc-relayer-collector` |
| URL pública | `https://data.relayers.cumulo.com.es/activity.json` |

---

## Wallets configuradas

| Key | Chain | Network | Dirección |
|---|---|---|---|
| `celestia_mainnet` | celestia | mainnet | `celestia1ahzs5c3g3z9u9wxwxmcmqgu6htagp4md627uas` |
| `cosmoshub_mainnet` | cosmoshub-4 | mainnet | `cosmos1t24lx6zfx7hqrexppk6gh2yavytv39xqfg5u9f` |
| `xrplevm_mainnet` | xrplevm_1440000-1 | mainnet | `ethm1jl3w0f8r6688ghd30he8ddjtnmtuevkvtjwj6r` |
| `injective_mainnet` | injective-1 | mainnet | `inj1jxr2se9ut4yxqz4jemz3qn9knkaklwngvc3c3w` |
| `seda_mainnet` | seda-1 | mainnet | `seda10l7d6fe3l68zghf6fyrpj5xqpj748sfg6jszcv` |
| `osmosis_mainnet` | osmosis-1 | mainnet | `osmo1djcckm2zw4xyz2lt6yg6euhgnrwp29k38n2l86` |
| `dymension_mainnet` | dymension_1100-1 | mainnet | `dym1qqev5krh9ge9tctc4gkfs6l54anuunzjngwkcl` |
| `celestia_testnet` | mocha-4 | testnet | `celestia1mlf0e4cx657a2mkt9t9clcw78zhqut4629v3s7` |
| `cosmoshub_testnet` | provider | testnet | `cosmos1qqsrv08cvqpxcctmr7pyy3c5g3ukvurktejy2q` |

---

## Añadir una nueva wallet

Editar el array `WALLETS` en el script:

```bash
nano /opt/ibc-relayer-collector/collector-relayers.js
```

Añadir una entrada con este formato:

```javascript
{
  key:     "chain_mainnet",          // identificador único, snake_case
  chain:   "chain-id",               // chain_id exacto de la red
  network: "mainnet",                // "mainnet" o "testnet"
  label:   "Chain Mainnet",          // nombre mostrado en el dashboard
  address: "addr1...",               // dirección del wallet relayer
  rest:    "https://api.chain.com",  // endpoint REST (preferiblemente propio de Cumulo)
  explorer_tx: "https://mintscan.io/chain/tx/",  // null si no hay explorer
  denom:   "utoken",                 // denom nativo (el que paga fees)
  denom_display: "TOKEN",            // símbolo para mostrar
  exp:     1e6,                      // 1e6 para 6 decimales, 1e18 para 18 decimales
},
```

**Decimales por chain:**
- `1e6` → Celestia (TIA), Cosmos Hub (ATOM), Osmosis (OSMO)
- `1e18` → Injective (INJ), SEDA (SEDA), XRPL EVM (XRP), Dymension (DYM)

Reiniciar el servicio tras guardar:

```bash
sudo systemctl restart ibc-relayer-collector
systemctl status ibc-relayer-collector
```

Verificar que la nueva wallet aparece en los logs:

```bash
journalctl -u ibc-relayer-collector -n 50
```

---

## Comandos útiles

```bash
# Ver estado del servicio
systemctl status ibc-relayer-collector

# Ejecutar una vez manualmente (sin tocar el servicio)
node /opt/ibc-relayer-collector/collector-relayers.js

# Ver logs en tiempo real
journalctl -u ibc-relayer-collector -f

# Ver últimas líneas del JSON generado
tail -c 500 /var/lib/cumulo-ibc-relayer/activity.json

# Reiniciar
sudo systemctl restart ibc-relayer-collector
```

---

## Funcionamiento

- Corre en modo `--watch`, ejecutándose cada **1 hora**
- En cada ejecución descarga txs nuevas desde la última altura conocida (incremental)
- Escribe atómicamente (`activity.json.tmp` → `activity.json`) para evitar lecturas parciales
- El dashboard lo consume vía `activity_proxy.php` con caché de 5 minutos

## Tipos de tx recogidos

| Tipo | Categoría |
|---|---|
| MsgRecvPacket | relay |
| MsgAcknowledgement | relay |
| MsgTimeout / MsgTimeoutOnClose | relay |
| MsgUpdateClient | maintenance |
| MsgCreateClient | maintenance |
