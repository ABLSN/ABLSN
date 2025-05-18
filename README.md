import logging
import asyncio
import json
import requests
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from solana.rpc.websocket_api import connect
from sklearn.ensemble import IsolationForest
import psycopg2
import redis
from telegram import Bot
from typing import Dict, List
from datetime import datetime

# Configuration
logging.basicConfig(level=logging.INFO)
app = FastAPI()

# Clés API et configuration
CONFIG = {
    "gmgn_api_key": "votre_gmgn_api_key",
    "rugcheck_api_key": "votre_rugcheck_api_key",
    "dexscreener_api_key": "votre_dexscreener_api_key",
    "pocket_universe_api_key": "votre_pocket_universe_api_key",
    "toxisolbot_api_token": "votre_toxisolbot_api_token",
    "solana_rpc": "wss://api.mainnet-beta.solana.com",
    "telegram_chat_id": "votre_chat_id",
    "min_rugcheck_score": 80,
    "max_concentration": 0.5,
    "min_liquidity": 10000,
    "min_token_age": 24,
    "min_volume": 50000,
    "require_locked_liquidity": True,
}

# Connexions
conn = psycopg2.connect("dbname=bot_db user=postgres password=secret")
cursor = conn.cursor()
redis_client = redis.Redis(host="localhost", port=6379, db=0)
telegram_bot = Bot(token=CONFIG["toxisolbot_api_token"])

# Modèles
class TradingFilter(BaseModel):
    min_liquidity: float
    min_token_age: int
    min_volume: float
    require_locked_liquidity: bool

class TokenAnalysisRequest(BaseModel):
    token_address: str
    filters: TradingFilter
    pocket_universe_api_key: str = None

# Initialisation de la base de données
cursor.execute("""
    CREATE TABLE IF NOT EXISTS blacklist (
        address TEXT PRIMARY KEY,
        type TEXT,
        reason TEXT,
        added_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
    CREATE TABLE IF NOT EXISTS tokens (
        address TEXT PRIMARY KEY,
        metadata JSONB
    );
    CREATE TABLE IF NOT EXISTS alerts (
        id SERIAL PRIMARY KEY,
        alert_type TEXT,
        message TEXT,
        token_address TEXT,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
    CREATE TABLE IF NOT EXISTS security_checks (
        id SERIAL PRIMARY KEY,
        token_address TEXT,
        rugcheck_score INTEGER,
        is_concentrated BOOLEAN,
        is_safe BOOLEAN,
        checked_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
    CREATE TABLE IF NOT EXISTS volume_checks (
        id SERIAL PRIMARY KEY,
        token_address TEXT,
        volume_usd FLOAT,
        is_fake BOOLEAN,
        source TEXT,
        checked_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
""")
conn.commit()

# Fonctions utilitaires
async def save_alert(alert_type: str, message: str, token_address: str):
    cursor.execute(
        "INSERT INTO alerts (alert_type, message, token_address) VALUES (%s, %s, %s)",
        (alert_type, message, token_address)
    )
    conn.commit()
    await telegram_bot.send_message(chat_id=CONFIG["telegram_chat_id"], text=message)

def cache_data(key: str, data: dict, ttl: int = 3600):
    redis_client.setex(key, ttl, json.dumps(data))

def get_cached_data(key: str) -> dict:
    data = redis_client.get(key)
    return json.loads(data) if data else None

# Collecte de données
async def fetch_gmgn_data(token_address: str) -> Dict:
    try:
        url = f"https://api.gmgn.ai/v1/tokens/{token_address}"
        headers = {"Authorization": f"Bearer {CONFIG['gmgn_api_key']}"}
        response = requests.get(url, headers=headers, timeout=10)
        response.raise_for_status()
        return response.json()
    except Exception as e:
        logging.error(f"Erreur GM Sergio GMGN: {e}")
        return {}

async def fetch_dexscreener_data(token_address: str) -> Dict:
    try:
        url = f"https://api.dexscreener.com/latest/dex/tokens/{token_address}"
        response = requests.get(url, timeout=10)
        response.raise_for_status()
        data = response.json()
        pair = data.get("pairs", [])[0] if data.get("pairs") else {}
        return {
            "address": token_address,
            "volume_24h": pair.get("volume", {}).get("h24", 0),
            "price_change_24h": pair.get("priceChange", {}).get("h24", 0),
            "makers": pair.get("txns", {}).get("h24", {}).get("buys", 0) + pair.get("txns", {}).get("h24", {}).get("sells", 0),
            "liquidity_usd": pair.get("liquidity", {}).get("usd", 0)
        }
    except Exception as e:
        logging.error(f"Erreur Dexscreener: {e}")
        return {}

async def fetch_holders_data(token_address: str) -> Dict:
    try:
        url = f"https://public-api.solscan.io/token/holders?tokenAddress={token_address}"
        response = requests.get(url, timeout=10)
        response.raise_for_status()
        data = response.json()
        return {
            "total_supply": data.get("totalSupply", 1),
            "top_holders": [{"amount": h["amount"]} for h in data.get("holders", [])[:10]]
        }
    except Exception as e:
        logging.error(f"Erreur Solscan holders: {e}")
        return {"total_supply": 1, "top_holders": []}

async def fetch_liquidity_data(token_address: str) -> Dict:
    # Hypothétique, à adapter selon l'API disponible
    return {"is_locked": True}  # Simulation

async def fetch_solscan_data(token_address: str) -> Dict:
    # Hypothétique, à adapter
    return {"is_verified": True}  # Simulation

# Vérifications de sécurité
async def check_blacklist(address: str) -> Dict:
    cursor.execute("SELECT * FROM blacklist WHERE address = %s", (address,))
    result = cursor.fetchone()
    return {"is_blacklisted": bool(result), "details": result}

def check_rugcheck(token_address: str, min_score: int = CONFIG["min_rugcheck_score"]) -> bool:
    try:
        url = f"https://api.rugcheck.xyz/v1/token/{token_address}"
        headers = {"Authorization": f"Bearer {CONFIG['rugcheck_api_key']}"}
        response = requests.get(url, headers=headers, timeout=10)
        response.raise_for_status()
        data = response.json()
        score = data.get("risk_score", 0)
        issues = data.get("issues", [])
        return score >= min_score and not any(i["critical"] for i in issues)
    except Exception as e:
        logging.error(f"Erreur RugCheck: {e}")
        return False

async def check_token_concentration(token_address: str, max_concentration: float = CONFIG["max_concentration"], max_holders: int = 10) -> bool:
    try:
        holders_data = await fetch_holders_data(token_address)
        total_supply = holders_data.get("total_supply", 1)
        top_holders = holders_data.get("top_holders", [])[:max_holders]
        top_holders_supply = sum(holder["amount"] for holder in top_holders)
        concentration = top_holders_supply / total_supply
        return concentration <= max_concentration
    except Exception as e:
        logging.error(f"Erreur concentration: {e}")
        return False

async def is_correct_contract(token_address: str) -> bool:
    if not check_rugcheck(token_address):
        return False
    liquidity_data = await fetch_liquidity_data(token_address)
    if not liquidity_data.get("is_locked", False):
        return False
    solscan_data = await fetch_solscan_data(token_address)
    if not solscan_data.get("is_verified", False):
        return False
    return True

async def check_trading_filters(token_address: str, filters: TradingFilter) -> bool:
    token_data = await fetch_dexscreener_data(token_address)
    liquidity_data = await fetch_liquidity_data(token_address)
    return (
        token_data.get("liquidity_usd", 0) >= filters.min_liquidity and
        token_data.get("age_hours", 0) >= filters.min_token_age and
        token_data.get("volume_24h", 0) >= filters.min_volume and
        (not filters.require_locked_liquidity or liquidity_data.get("is_locked", False))
    )

def is_fake_volume(token_data: Dict, volume_threshold: float = 100000, min_makers: int = 10) -> bool:
    volume_usd = token_data.get("volume_24h", 0)
    price_change = token_data.get("price_change_24h", 0)
    makers = token_data.get("makers", 0)
    if volume_usd > volume_threshold:
        if abs(price_change) < 5 or makers < min_makers:
            return True
    return False

def check_volume_pocket_universe(token_address: str, volume_usd: float, api_key: str) -> bool:
    try:
        url = "https://api.pocketuniverse.com/v1/volume/check"
        headers = {"Authorization": f"Bearer {api_key}"}
        params = {"token_address": token_address, "volume_usd": volume_usd}
        response = requests.get(url, headers=headers, params=params, timeout=10)
        response.raise_for_status()
        return response.json().get("is_fake", False)
    except Exception as e:
        logging.error(f"Erreur Pocket Universe: {e}")
        return True

# Analyse
def detect_pump(token_data: Dict, threshold: float = 5.0) -> bool:
    price_change = token_data.get("price_change_24h", 0)
    return price_change > threshold

def detect_scam(token_data: Dict) -> bool:
    # Simulation avec IsolationForest
    data = [[token_data.get("volume_24h", 0)]]
    model = IsolationForest(contamination=0.2)
    model.fit(data)
    return model.predict(data)[0] == -1

# ToxiSolBot
async def execute_toxisol_trade(token_address: str, action: str, amount: float):
    """
    Exécute une transaction via ToxiSolBot.
    :param action: "buy" ou "sell"
    :param amount: Montant en SOL
    """
    try:
        url = "https://api.toxisolbot.com/v1/trade"  # Hypothétique
        headers = {"Authorization": f"Bearer {CONFIG['toxisolbot_api_token']}"}
        payload = {
            "token_address": token_address,
            "action": action,
            "amount": amount,
            "slippage": 0.11  # 11% par défaut, ajustable
        }
        response = requests.post(url, headers=headers, json=payload, timeout=10)
        response.raise_for_status()
        trade_id = response.json().get("trade_id")
        await save_alert(
            f"trade_{action}",
            f"Trade {action} exécuté pour {token_address} (ID: {trade_id})",
            token_address
        )
        return trade_id
    except Exception as e:
        logging.error(f"Erreur ToxiSolBot trade: {e}")
        await save_alert("trade_error", f"Erreur trade {action} pour {token_address}: {e}", token_address)
        return None

# Pipeline principal
@app.post("/analyze_token")
async def analyze_token(request: TokenAnalysisRequest):
    token_address = request.token_address
    filters = request.filters
    api_key = request.pocket_universe_api_key or CONFIG["pocket_universe_api_key"]

    # Étape 1 : Liste noire
    blacklist_check = await check_blacklist(token_address)
    if blacklist_check["is_blacklisted"]:
        await save_alert("blacklisted", f"Token {token_address} est blacklisté", token_address)
        raise HTTPException(status_code=400, detail="Token blacklisté")

    # Étape 2 : Contrat
    if not await is_correct_contract(token_address):
        cursor.execute(
            "INSERT INTO blacklist (address, type, reason) VALUES (%s, %s, %s) ON CONFLICT DO NOTHING",
            (token_address, "token", "Contrat non sûr")
        )
        conn.commit()
        await save_alert("unsafe_contract", f"Token {token_address} : Contrat non sûr", token_address)
        raise HTTPException(status_code=400, detail="Contrat non sûr")

    # Étape 3 : Concentration
    is_concentrated = not await check_token_concentration(token_address)
    if is_concentrated:
        cursor.execute(
            "INSERT INTO blacklist (address, type, reason) VALUES (%s, %s, %s) ON CONFLICT DO NOTHING",
            (token_address, "token", "Offre trop concentrée")
        )
        conn.commit()
        await save_alert("concentrated_supply", f"Token {token_address} : Offre trop concentrée", token_address)
        raise HTTPException(status_code=400, detail="Offre trop concentrée")

    # Étape 4 : Filtres
    if not await check_trading_filters(token_address, filters):
        await save_alert("filtered", f"Token {token_address} ne passe pas les filtres", token_address)
        raise HTTPException(status_code=400, detail="Token filtré")

    # Étape 5 : Volume fictif
    token_data = await fetch_gmgn_data(token_address)
    token_data.update(await fetch_dexscreener_data(token_address))
    is_fake = is_fake_volume(token_data) or check_volume_pocket_universe(token_address, token_data["volume_24h"], api_key)
    if is_fake:
        cursor.execute(
            "INSERT INTO blacklist (address, type, reason) VALUES (%s, %s, %s) ON CONFLICT DO NOTHING",
            (token_address, "token", "Volume fictif")
        )
        cursor.execute(
            "INSERT INTO volume_checks (token_address, volume_usd, is_fake, source) VALUES (%s, %s, %s, %s)",
            (token_address, token_data["volume_24h"], True, "algo_pocket")
        )
        conn.commit()
        await save_alert("fake_volume", f"Volume fictif détecté pour {token_address}", token_address)
        raise HTTPException(status_code=400, detail="Volume fictif")

    # Étape 6 : Analyse
    rugcheck_score = 85  # Simulation
    if detect_pump(token_data):
        await save_alert("pump", f"Pump détecté sur {token_address}: +{token_data['price_change_24h']}%", token_address)
        # Transaction automatique
        trade_id = await execute_toxisol_trade(token_address, "buy", 0.1)  # Exemple : 0.1 SOL
        if trade_id:
            # Planifier une vente après un seuil de profit
            if token_data["price_change_24h"] > 10:  # Exemple : +10%
                await execute_toxisol_trade(token_address, "sell", 0.1)
    if detect_scam(token_data):
        await save_alert("scam", f"Arnaque détectée sur {token_address}", token_address)

    # Étape 7 : Stockage
    cursor.execute(
        "INSERT INTO tokens (address, metadata) VALUES (%s, %s) ON CONFLICT DO NOTHING",
        (token_address, json.dumps(token_data))
    )
    cursor.execute(
        "INSERT INTO security_checks (token_address, rugcheck_score, is_concentrated, is_safe) VALUES (%s, %s, %s, %s)",
        (token_address, rugcheck_score, is_concentrated, True)
    )
    conn.commit()
    cache_data(f"token:{token_address}", token_data)

    return {"status": "safe_for_trading", "data": token_data}

# Surveillance Solana en temps réel
async def listen_solana_transactions():
    async with connect(CONFIG["solana_rpc"]) as websocket:
        await websocket.logs_subscribe()
        async for msg in websocket:
            token_address = msg.get("result", {}).get("value", {}).get("address", "")
            if token_address:
                await analyze_token(TokenAnalysisRequest(
                    token_address=token_address,
                    filters=TradingFilter(
                        min_liquidity=CONFIG["min_liquidity"],
                        min_token_age=CONFIG["min_token_age"],
                        min_volume=CONFIG["min_volume"],
                        require_locked_liquidity=CONFIG["require_locked_liquidity"]
                    )
                ))

# Démarrage
if __name__ == "__main__":
    import uvicorn
    asyncio.run(listen_solana_transactions())
    uvicorn.run(app, host="0.0.0.0", port=8000)
