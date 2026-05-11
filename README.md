#!/usr/bin/env python3
“””
MHI Forex Tracker Bot — Telegram
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Detecta señales MHI (minoría de velas) en pares forex.
Alerta tras 2 ciclos fallidos consecutivos para entrar en el 3er ciclo.
Registra el resultado del 3er ciclo y te lo notifica.

Instalación:
pip install python-telegram-bot requests

Uso:
python mhi_bot.py
“””

import asyncio
import logging
import requests
import time
from datetime import datetime
from collections import defaultdict
from telegram import Bot
from telegram.error import TelegramError

# ── Configuración ─────────────────────────────────────────────────────────────

TOKEN   = “8699137663:AAEMr8MlsetcVJmcNL2QJQ2M3gG2jwczp_o”
CHAT_ID = “1450779260”

PAIRS = [
“EUR/USD”, “GBP/USD”, “USD/JPY”,
“AUD/USD”, “USD/CAD”, “EUR/GBP”,
“GBP/JPY”, “NZD/USD”
]

# Símbolos en formato para la API (Alpha Vantage o simulado)

PAIR_SYMBOLS = {
“EUR/USD”: “EURUSD”, “GBP/USD”: “GBPUSD”, “USD/JPY”: “USDJPY”,
“AUD/USD”: “AUDUSD”, “USD/CAD”: “USDCAD”, “EUR/GBP”: “EURGBP”,
“GBP/JPY”: “GBPJPY”, “NZD/USD”: “NZDUSD”
}

CHECK_INTERVAL = 60  # segundos entre cada comprobación (1 minuto = velas de 1 min)

logging.basicConfig(
format=”%(asctime)s [%(levelname)s] %(message)s”,
level=logging.INFO,
datefmt=”%H:%M:%S”
)
log = logging.getLogger(**name**)

# ── Obtener velas (simulado — reemplaza con API real si tienes clave) ──────────

def get_candles(pair: str, n: int = 5) -> list[dict]:
“””
Devuelve las últimas N velas de 1 minuto para el par dado.

```
MODO SIMULADO: genera velas aleatorias realistas.
Para datos reales, usa Alpha Vantage, Twelve Data, o el broker API.

Para activar datos reales con Alpha Vantage (gratis):
  1. Regístrate en https://www.alphavantage.co/support/#api-key
  2. Reemplaza la función por la versión comentada más abajo.
"""
import random
base_prices = {
    "EUR/USD": 1.0850, "GBP/USD": 1.2650, "USD/JPY": 149.50,
    "AUD/USD": 0.6520, "USD/CAD": 1.3680, "EUR/GBP": 0.8580,
    "GBP/JPY": 189.20, "NZD/USD": 0.5980,
}
price = base_prices.get(pair, 1.0)
candles = []
for _ in range(n):
    vol = price * 0.0012
    move = (random.random() - 0.497) * vol
    close = max(price + move, 0.0001)
    candles.append({"open": price, "close": close, "bull": close >= price})
    price = close
return candles
```

# ── Versión con Alpha Vantage (descomenta si tienes API key) ──────────────────

# AV_API_KEY = “TU_API_KEY_AQUI”

# 

# def get_candles_real(pair: str, n: int = 5) -> list[dict]:

# symbol = PAIR_SYMBOLS.get(pair, “EURUSD”)

# url = (

# f”https://www.alphavantage.co/query”

# f”?function=FX_INTRADAY&from_symbol={symbol[:3]}&to_symbol={symbol[3:]}”

# f”&interval=1min&outputsize=compact&apikey={AV_API_KEY}”

# )

# try:

# r = requests.get(url, timeout=10)

# data = r.json().get(“Time Series FX (1min)”, {})

# candles = []

# for ts in sorted(data.keys(), reverse=True)[:n]:

# c = data[ts]

# op = float(c[“1. open”]); cl = float(c[“4. close”])

# candles.append({“open”: op, “close”: cl, “bull”: cl >= op})

# return list(reversed(candles))

# except Exception as e:

# log.error(f”Error API {pair}: {e}”)

# return []

# ── Señal MHI: minoría de las últimas 3 velas ────────────────────────────────

def mhi_signal(candles: list[dict]) -> str | None:
“””
MHI = Minoría:
- 2 alcistas + 1 bajista → CALL  (minoría es bajista)
- 2 bajistas + 1 alcista → PUT   (minoría es alcista)
- 3 iguales              → None  (sin señal)
“””
if len(candles) < 3:
return None
last3 = candles[-3:]
bulls = sum(1 for c in last3 if c[“bull”])
if bulls == 1:
return “CALL”
if bulls == 2:
return “PUT”
return None

# ── Estado por par ────────────────────────────────────────────────────────────

class PairState:
def **init**(self, pair: str):
self.pair          = pair
self.phase         = “IDLE”   # IDLE | ENTRY | MG1 | MG2 | WAITING_3RD
self.signal        = None
self.failed_cycles = 0        # ciclos fallidos consecutivos
self.in_third      = False    # estamos rastreando el 3er ciclo
self.third_ops     = []       # ops del 3er ciclo
self.third_phase   = “IDLE”   # fase dentro del 3er ciclo
self.third_signal  = None
self.stats         = {“alerts”: 0, “third_wins”: 0, “third_losses”: 0}
self.candles       = []

```
def reset_cycle(self):
    self.phase  = "IDLE"
    self.signal = None

def reset_third(self):
    self.in_third    = False
    self.third_ops   = []
    self.third_phase = "IDLE"
    self.third_signal = None

@property
def third_win_rate(self) -> str:
    total = self.stats["third_wins"] + self.stats["third_losses"]
    if total == 0:
        return "Sin datos aún"
    rate = (self.stats["third_wins"] / total) * 100
    return f"{rate:.0f}% ({self.stats['third_wins']}W / {self.stats['third_losses']}L)"
```

# ── Emojis ────────────────────────────────────────────────────────────────────

SIGNAL_EMOJI = {“CALL”: “📈”, “PUT”: “📉”}
PHASE_EMOJI  = {“ENTRY”: “1️⃣”, “MG1”: “2️⃣”, “MG2”: “3️⃣”}

# ── Bot principal ─────────────────────────────────────────────────────────────

class MHIBot:
def **init**(self):
self.bot    = Bot(token=TOKEN)
self.states = {pair: PairState(pair) for pair in PAIRS}

```
async def send(self, text: str):
    try:
        await self.bot.send_message(
            chat_id=CHAT_ID,
            text=text,
            parse_mode="Markdown"
        )
        log.info(f"Mensaje enviado: {text[:60]}...")
    except TelegramError as e:
        log.error(f"Error Telegram: {e}")

async def process_pair(self, st: PairState):
    candles = get_candles(st.pair, n=10)
    if not candles:
        return
    st.candles = candles
    last = candles[-1]
    now  = datetime.now().strftime("%H:%M")

    # ── Rastreo del 3er ciclo (tiene prioridad) ──────────────────────────
    if st.in_third:
        await self._process_third_cycle(st, last, now)
        return

    # ── Ciclo normal (detectar señal y rastrear) ─────────────────────────
    if st.phase == "IDLE":
        sig = mhi_signal(candles)
        if sig:
            st.signal = sig
            st.phase  = "ENTRY"
            log.info(f"{st.pair} → señal MHI {sig} detectada")
        return

    # Resolver operación abierta
    if st.phase in ("ENTRY", "MG1", "MG2"):
        won = last["bull"] if st.signal == "CALL" else not last["bull"]
        phase_name = st.phase

        if won:
            log.info(f"{st.pair} {phase_name} → GANADA ✓")
            st.failed_cycles = 0
            st.reset_cycle()
        else:
            log.info(f"{st.pair} {phase_name} → PERDIDA ✗")
            if phase_name == "ENTRY":
                st.phase  = "MG1"
                st.signal = mhi_signal(candles) or st.signal
            elif phase_name == "MG1":
                st.phase  = "MG2"
                st.signal = mhi_signal(candles) or st.signal
            elif phase_name == "MG2":
                # Ciclo completo fallido
                st.failed_cycles += 1
                log.info(f"{st.pair} → ciclo fallido #{st.failed_cycles}")

                if st.failed_cycles >= 2:
                    # 🚨 ALERTA — 2 ciclos fallidos consecutivos
                    st.stats["alerts"] += 1
                    st.failed_cycles = 0
                    sig3 = mhi_signal(candles)
                    st.in_third      = True
                    st.third_phase   = "ENTRY"
                    st.third_signal  = sig3
                    st.third_ops     = []
                    st.reset_cycle()

                    emoji = SIGNAL_EMOJI.get(sig3, "⚠️") if sig3 else "⚠️"
                    msg = (
                        f"🚨 *ALERTA MHI — {st.pair}* 🚨\n"
                        f"━━━━━━━━━━━━━━━━━━━\n"
                        f"2 ciclos fallidos seguidos.\n"
                        f"*Entra en el 3er ciclo ahora.*\n\n"
                        f"{emoji} Señal MHI: *{sig3 or 'Revisa el gráfico'}*\n"
                        f"🕐 {now}\n\n"
                        f"📊 Tasa de éxito 3er ciclo: {st.third_win_rate}"
                    )
                    await self.send(msg)
                else:
                    st.reset_cycle()

async def _process_third_cycle(self, st: PairState, last: dict, now: str):
    """Rastrea el resultado del 3er ciclo y notifica."""
    if st.third_phase == "IDLE":
        return

    if st.third_phase in ("ENTRY", "MG1", "MG2"):
        won = last["bull"] if st.third_signal == "CALL" else not last["bull"]
        phase_name = st.third_phase
        st.third_ops.append({"phase": phase_name, "won": won})

        if won:
            st.stats["third_wins"] += 1
            msg = (
                f"✅ *3er CICLO GANADO — {st.pair}*\n"
                f"━━━━━━━━━━━━━━━━━━━\n"
                f"Operación: *{phase_name}*\n"
                f"Señal: {SIGNAL_EMOJI.get(st.third_signal,'')} {st.third_signal}\n"
                f"🕐 {now}\n\n"
                f"📊 Tasa de éxito 3er ciclo: {st.third_win_rate}"
            )
            await self.send(msg)
            st.reset_third()
            st.reset_cycle()

        else:
            if phase_name == "ENTRY":
                st.third_phase  = "MG1"
                st.third_signal = mhi_signal(st.candles) or st.third_signal
                await self.send(
                    f"⚠️ *{st.pair} — 3er ciclo: ENTRY perdida*\n"
                    f"Abre *MG1* {SIGNAL_EMOJI.get(st.third_signal,'')} {st.third_signal}\n"
                    f"🕐 {now}"
                )
            elif phase_name == "MG1":
                st.third_phase  = "MG2"
                st.third_signal = mhi_signal(st.candles) or st.third_signal
                await self.send(
                    f"⚠️ *{st.pair} — 3er ciclo: MG1 perdida*\n"
                    f"Abre *MG2* {SIGNAL_EMOJI.get(st.third_signal,'')} {st.third_signal}\n"
                    f"🕐 {now}"
                )
            elif phase_name == "MG2":
                # 3er ciclo completo fallido
                st.stats["third_losses"] += 1
                msg = (
                    f"❌ *3er CICLO FALLIDO — {st.pair}*\n"
                    f"━━━━━━━━━━━━━━━━━━━\n"
                    f"El ciclo completo se ha perdido.\n"
                    f"🕐 {now}\n\n"
                    f"📊 Tasa de éxito 3er ciclo: {st.third_win_rate}"
                )
                await self.send(msg)
                st.reset_third()
                st.reset_cycle()

async def run(self):
    await self.send(
        "🤖 *MHI Forex Tracker iniciado*\n"
        "━━━━━━━━━━━━━━━━━━━\n"
        "Monitoreando pares:\n"
        + "\n".join(f"• {p}" for p in PAIRS) +
        f"\n\nTe avisaré cuando 2 ciclos fallen seguidos.\n"
        f"Intervalo: {CHECK_INTERVAL}s (velas 1 min)\n\n"
        f"_Escribe /stop para detener_"
    )
    log.info("Bot iniciado. Monitoreando mercado...")

    while True:
        log.info(f"── Comprobando {len(PAIRS)} pares ──")
        for pair in PAIRS:
            try:
                await self.process_pair(self.states[pair])
            except Exception as e:
                log.error(f"Error procesando {pair}: {e}")
            await asyncio.sleep(0.3)  # pequeña pausa entre pares

        await asyncio.sleep(CHECK_INTERVAL)
```

# ── Entry point ───────────────────────────────────────────────────────────────

if **name** == “**main**”:
bot = MHIBot()
try:
asyncio.run(bot.run())
except KeyboardInterrupt:
log.info(“Bot detenido.”)