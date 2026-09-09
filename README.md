# vfat-position-watchdog

Demonio de terminal (Linux, Python 3.8+, **solo librería estándar**) que vigila
indefinidamente las **posiciones LP abiertas de una wallet** mediante el MCP
público de [vfat](https://vfat.io/mcp#tools) y envía un aviso por **Telegram**
en cada ejecución en la que alguna posición de liquidez concentrada esté
**fuera de rango**.

```
┌────────────┐   cada CHECK_INTERVAL_SECONDS    ┌──────────────┐
│ vfat_watchdog.py ─────────────────────────────▶ │ MCP vfat.io  │
│ (demonio)  │   tools/call:                     │  (HTTP)      │
│            │   get_wallet_portfolio             └──────────────┘
│            │        │  tickLow/tickUp vs tick actual del pool
│            │        ▼
│            │   ¿alguna posición fuera de rango? ──▶ 🚨 Telegram
└────────────┘
```

## Cómo decide si una posición está fuera de rango

El tool `get_wallet_portfolio` del MCP devuelve, para cada posición NFT de
liquidez concentrada (Uniswap V3, Aerodrome Slipstream, etc.), el rango de la
posición (`nft.tickLow` / `nft.tickUp`) y el tick actual del pool (`tick`). La
posición está **en rango** si:

```
tickLow ≤ tick ≤ tickUp
```

Las posiciones sin rango (pares clásicos sin liquidez concentrada) se
registran en el log como *no vigilables* pero no generan avisos. El precio
humano se recalcula como `1.0001^tick · 10^(dec0−dec1)` para mostrar el
porcentaje de distancia al límite del rango.

## Instalación

```bash
# 1) Python 3.8+ (no hay dependencias externas)
python3 --version

# 2) Configuración
cp .env.example .env
nano .env          # wallet, bot de Telegram, etc.

# 3) Dar permisos de ejecución
chmod +x vfat_watchdog.py
```

### Crear el bot de Telegram

1. Habla con [@BotFather](https://t.me/BotFather) → `/newbot` → copia el token.
2. Escribe un mensaje a tu bot (o al canal/grupo donde lo añadas).
3. Obtén el `chat_id` abriendo `https://api.telegram.org/bot<TOKEN>/getUpdates`
   y copiando `result[].message.chat.id`.

## Uso

```bash
./vfat_watchdog.py                    # demonio: bucle infinito cada 3 min
./vfat_watchdog.py --env /ruta/.env   # fichero de configuración alternativo
./vfat_watchdog.py --interval 60      # pisa CHECK_INTERVAL_SECONDS
./vfat_watchdog.py --once             # una comprobación y sale (ideal para cron)
./vfat_watchdog.py --once --dry-run   # igual pero sin enviar Telegram (pruebas)
./vfat_watchdog.py --verbose          # logs de depuración
```

Parar el demonio: `Ctrl+C` o `SIGTERM` (termina limpiamente tras el ciclo en
curso). En segundo plano: `nohup ./vfat_watchdog.py >/dev/null 2>&1 &`.

### Primeras pruebas recomendadas

```bash
./vfat_watchdog.py --once --dry-run   # ¿lee bien las posiciones?
./vfat_watchdog.py --once             # ¿llega el mensaje de arranque a Telegram?
```

## Configuración (`.env`)

| Variable | Por defecto | Descripción |
|---|---|---|
| `WALLET_ADDRESS` | — | **Obligatoria.** Wallet EVM a vigilar (admite lista separada por comas, máx. 5). |
| `TELEGRAM_BOT_TOKEN` | — | Obligatorio salvo `--dry-run`. Token de @BotFather. |
| `TELEGRAM_CHAT_ID` | — | Obligatorio salvo `--dry-run`. Chat/canal destino. |
| `CHECK_INTERVAL_SECONDS` | `180` | Segundos entre comprobaciones (3 min). |
| `CHAIN_IDS` | *(todas)* | Filtro opcional de chains, p. ej. `8453,42161`. |
| `ALERT_ONLY_CHANGES` | `false` | `true` = avisa solo al *cambiar* a fuera de rango en vez de en cada ejecución. |
| `ALERT_ON_RECOVERY` | `true` | Avisa también cuando la posición vuelve a entrar en rango. |
| `NOTIFY_ON_START` | `true` | Mensaje de bienvenida al arrancar. |
| `NOTIFY_ON_ERROR` | `true` | Avisa si una comprobación falla y cuando se recupera. |
| `HEARTBEAT_HOURS` | `0` | Resumen periódico "todo en orden" (0 = desactivado). |
| `LOG_FILE` | *(stdout)* | Fichero adicional de log. |
| `MCP_URL` | `https://mcp.vfat.io/mcp` | Endpoint del MCP (no requiere API key). |

## Ejecutar como servicio (systemd)

```bash
sudo cp vfat-watchdog.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now vfat-watchdog
journalctl -u vfat-watchdog -f          # ver logs
```

## Ejecutar con cron (alternativa a systemd)

```cron
*/3 * * * * cd /home/ubuntu/CryptoRobotFlash && ./vfat_watchdog.py --once >> watchdog.cron.log 2>&1
```

## Pruebas

`tests/test_watchdog.py` verifica con datos sintéticos el cálculo de rango, el
precio humano (`1.0001^tick · 10^(dec0−dec1)`), el filtrado por wallet/chain,
los avisos de salida y de retorno a rango, y el modo `ALERT_ONLY_CHANGES`:

```bash
python3 tests/test_watchdog.py
```

## Notas

- **Rate limit**: el MCP de vfat limita las peticiones; el demonio hace 1
  llamada cada 3 minutos (hasta 5 wallets por llamada), muy por debajo del
  límite. Ante errores HTTP 429/5xx reintenta con backoff.
- **Errores transitorios**: si una comprobación falla (red, MCP caído) el
  demonio no muere: lo registra, avisa por Telegram y reintenta en el siguiente
  ciclo. Con `--once` el proceso termina con código 1 para que cron lo detecte.
- **Sin claves privadas**: solo se lee la wallet (dirección pública). El MCP de
  vfat devuelve datos y calldata sin firmar; el demonio nunca firma ni envía
  transacciones.
- El estado "en rango / fuera de rango" entre reinicios se mantiene en memoria;
  con `ALERT_ONLY_CHANGES=true`, tras reiniciar el demonio se avisa otra vez de
  las posiciones que sigan fuera de rango.
