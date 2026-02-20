# barres_au_sol — Data lake Parquet pour trading algorithmique

> Data cache Parquet pour backtesting (Dukascopy + CCXT)  
> Les barres sont dans au parquet. 🩵🎭

---

## 🎭 Pourquoi ce nom ?

**barres_au_sol** est un jeu de mots sur deux univers :

1. **Trading** : Les "barres" (candlesticks OHLC) sont stockées dans des fichiers **Parquet** (format Apache)
2. **Ballet** : Les "barres au sol" sont des exercices de danse classique (assouplissement, échauffement)

Ce projet alimente **Arabesque**, un système de trading dont les stratégies portent des noms de mouvements de danse :
- **Arabesque** (la stratégie principale)
- **Tombées** / **Envolées** (mouvements de mean-reversion et trend)

L'idée : les données de marché sont la "barre" à laquelle le système s'entraîne, avant de "danser" en live. 🩰

---

## Objectif

Constituer un **data lake local** en Parquet pour backtesting robuste :
- Source de vérité : barres **minute 1** (Dukascopy / CCXT Binance)
- Dérivés recalculables : **5m**, **1h**, **4h** (pour Backtrader, vectorbt, Arabesque)
- Téléchargements **incrémentaux** : relancer sans risque de doublons
- Orchestration via `instruments.csv` : 117 instruments (FX, indices, métaux, énergies, crypto)

---

## Architecture

```
barres_au_sol/
├── data/
│   ├── dukascopy/
│   │   ├── min1/          # Barres brutes 1 minute (source de vérité)
│   │   └── derived/       # Dérivés 5m/1h/4h
│   └── ccxt/
│       ├── min1/
│       └── derived/
├── docs/
│   └── INSTRUMENTS_STATUS.md  # Rapport FTMO/GFT
├── instruments.csv         # Configuration des 117 instruments
├── data_backends.py         # Abstractions Dukascopy + CCXT
├── data_orchestrator.py     # CLI principal
└── README.md
```

---

## Installation

### Prérequis

```bash
python --version  # 3.10+ requis
git --version
```

### Setup

```bash
cd ~/dev
git clone git@github.com:ashledombos/barres_au_sol.git
cd barres_au_sol

python3 -m venv .venv
source .venv/bin/activate

pip install pandas numpy pyarrow requests ccxt tqdm
```

**Optionnels** :
```bash
# Si tu utilises Backtrader
pip install backtrader

# Si tu fais du criblage vectoriel
pip install vectorbt
```

---

## Utilisation

### Téléchargement initial (2 ans de données)

```bash
# Tous les instruments (117) depuis 2024-01-01
python data_orchestrator.py \
  --start 2024-01-01 \
  --end 2026-02-20 \
  --sleep-ms 200 \
  --sleep-between 2
```

**Durée** : 30-60 minutes (dépend de la connexion)

### Mode plan (vérifier sans télécharger)

```bash
python data_orchestrator.py \
  --start 2024-01-01 \
  --end 2026-02-20 \
  --plan
```

Affiche :
- Quels fichiers Parquet existent
- Ce qui manque
- Couverture par instrument

### Télécharger un sous-ensemble

```bash
# Seulement les cryptos
python data_orchestrator.py \
  --start 2024-01-01 \
  --end 2026-02-20 \
  --filter "^(BTC|ETH|SOL|XRP)"

# Seulement les paires EUR
python data_orchestrator.py \
  --start 2024-01-01 \
  --end 2026-02-20 \
  --filter "^EUR"
```

### Recalculer les dérivés (sans download)

```bash
# Recalculer 5m, 1h, 4h depuis les min1 existants
python data_orchestrator.py \
  --start 2024-01-01 \
  --end 2026-02-20 \
  --derive-only

# Dérivés personnalisés
python data_orchestrator.py \
  --start 2024-01-01 \
  --end 2026-02-20 \
  --derive 15m 1h 1d
```

---

## Automatisation (cron vs systemd)

### Option 1 : cron (simple, universel)

**Avantages** :
- Simple à configurer
- Universel (fonctionne sur tout Linux/macOS)
- Pas de dépendance systemd

**Inconvénients** :
- Pas de gestion de retry si échec
- Logs basiques (redirection manuelle)
- Pas de dépendances (ex: attendre le réseau)

**Setup** :
```bash
crontab -e
```

Ajouter :
```cron
# Mise à jour quotidienne à 3h du matin
0 3 * * * cd /home/raphael/dev/barres_au_sol && .venv/bin/activate && python data_orchestrator.py --start 2024-01-01 --end $(date +\%Y-\%m-\%d) --sleep-ms 200 --sleep-between 2 >> logs/cron.log 2>&1
```

---

### Option 2 : systemd timer (recommandé)

✅ **Recommandation** : Utiliser systemd timer si disponible (Linux moderne).

**Avantages** :
- Gestion de retry automatique
- Logs structurés (`journalctl -u barres-au-sol`)
- Dépendances explicites (ex: attendre `network-online.target`)
- Visualisation statut (`systemctl status barres-au-sol.timer`)
- Pas de risque d'exécutions simultanées

**Setup** :

#### 1. Créer le service

Créer `~/.config/systemd/user/barres-au-sol.service` :

```ini
[Unit]
Description=barres_au_sol — Mise à jour données Parquet
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
WorkingDirectory=/home/raphael/dev/barres_au_sol
ExecStart=/home/raphael/dev/barres_au_sol/.venv/bin/python data_orchestrator.py --start 2024-01-01 --end %Y-%m-%d --sleep-ms 200 --sleep-between 2
StandardOutput=journal
StandardError=journal

# Retry si échec réseau
Restart=on-failure
RestartSec=300

[Install]
WantedBy=default.target
```

#### 2. Créer le timer

Créer `~/.config/systemd/user/barres-au-sol.timer` :

```ini
[Unit]
Description=barres_au_sol — Timer quotidien
Requires=barres-au-sol.service

[Timer]
OnCalendar=daily
OnCalendar=03:00
Persistent=true
RandomizedDelaySec=300

[Install]
WantedBy=timers.target
```

#### 3. Activer et démarrer

```bash
# Recharger systemd
systemctl --user daemon-reload

# Activer le timer (démarre au boot)
systemctl --user enable barres-au-sol.timer

# Démarrer le timer maintenant
systemctl --user start barres-au-sol.timer

# Vérifier le statut
systemctl --user status barres-au-sol.timer

# Voir les prochaines exécutions
systemctl --user list-timers
```

#### 4. Consulter les logs

```bash
# Logs du dernier run
journalctl --user -u barres-au-sol.service -n 100

# Logs en temps réel
journalctl --user -u barres-au-sol.service -f

# Logs depuis hier
journalctl --user -u barres-au-sol.service --since yesterday
```

#### 5. Tester manuellement

```bash
# Exécuter maintenant (sans attendre 3h)
systemctl --user start barres-au-sol.service

# Voir le statut
systemctl --user status barres-au-sol.service
```

---

## Fichier instruments.csv

Format :
```csv
ftmo_symbol,source,data_symbol,exchange,price_scale
EURUSD,dukascopy,EURUSD,,1e5
BTCUSD,ccxt,BTC/USDT,binance,
```

**Colonnes** :
- `ftmo_symbol` : Nom interne (utilisé par Arabesque)
- `source` : `dukascopy`, `ccxt`, ou `yahoo`
- `data_symbol` : Symbole source (ex: `EURUSD` pour Dukascopy, `BTC/USDT` pour Binance)
- `exchange` : Vide pour Dukascopy, `binance` pour CCXT
- `price_scale` : `1e5` pour FX (Dukascopy), vide pour crypto

**117 instruments configurés** :
- 47 paires FX (Dukascopy)
- 14 indices (Dukascopy)
- 9 métaux (Dukascopy)
- 4 énergies (Dukascopy)
- 7 commodities (Dukascopy)
- 31 cryptos (CCXT/Binance)

Voir `docs/INSTRUMENTS_STATUS.md` pour le détail.

---

## Troubleshooting

### Erreur "422 Invalid object" sur un instrument Dukascopy

**Cause** : Symbole non disponible sur Dukascopy ou nom incorrect.

**Solution** : Vérifier le nom dans `instruments.csv` (colonne `data_symbol`). Tester manuellement :
```bash
python data_orchestrator.py --start 2024-01-01 --end 2024-01-02 --filter "^NOMUSD$"
```

Si échec persist, commenter la ligne dans `instruments.csv`.

### Erreur "Rate limit exceeded" sur Binance

**Cause** : Trop de requêtes simultanées.

**Solution** : Augmenter les délais :
```bash
python data_orchestrator.py --start 2024-01-01 --end 2026-02-20 --sleep-ms 500 --sleep-between 5
```

### Parquet corrompu ou incomplet

**Solution** : Supprimer et retélécharger :
```bash
rm data/dukascopy/min1/EURUSD.parquet
rm data/dukascopy/derived/EURUSD_1h.parquet
python data_orchestrator.py --start 2024-01-01 --end 2026-02-20 --filter "^EURUSD$"
```

### Systemd timer ne se déclenche pas

**Vérifications** :
```bash
# Timer actif ?
systemctl --user is-enabled barres-au-sol.timer

# Prochaine exécution ?
systemctl --user list-timers | grep barres

# Logs systemd
journalctl --user -u barres-au-sol.timer
```

**Si le timer ne survit pas au reboot** :
```bash
# Activer linger (permet aux timers user de tourner sans session active)
sudo loginctl enable-linger $USER
```

---

## FAQ

### Pourquoi Dukascopy ET CCXT ?

- **Dukascopy** : Données FX/indices/commodities de qualité institutionnelle (utilisées par FTMO)
- **CCXT/Binance** : Crypto uniquement (Dukascopy ne propose pas de crypto)

### Pourquoi stocker en minute 1 ?

Les timeframes supérieurs (5m, 1h, 4h) sont **dérivés** des barres 1 minute. Cela permet :
- De recalculer n'importe quel timeframe sans retélécharger
- D'avoir la source de vérité la plus granulaire

### Quel espace disque requis ?

**Estimation pour 2 ans de données** :
- Dukascopy (80 instruments) : ~15 GB (min1) + ~3 GB (derived)
- CCXT (31 cryptos) : ~8 GB (min1) + ~2 GB (derived)
- **Total** : ~30 GB

### Peut-on utiliser d'autres exchanges que Binance ?

Oui. Modifier la colonne `exchange` dans `instruments.csv` :
```csv
BTCUSD,ccxt,BTC/USD,kraken,
```

CCXT supporte 100+ exchanges.

### Compatibilité avec d'autres frameworks ?

Les Parquets sont **framework-agnostic**. Exemples d'usage :

**Backtrader** :
```python
import pandas as pd
import backtrader as bt

df = pd.read_parquet("data/dukascopy/derived/EURUSD_1h.parquet")
feed = bt.feeds.PandasData(dataname=df)
```

**vectorbt** :
```python
import vectorbt as vbt
import pandas as pd

df = pd.read_parquet("data/dukascopy/derived/EURUSD_1h.parquet")
vbt.OHLCV.run(df).plot()
```

**Arabesque** :
```python
from arabesque.backtest.data import load_ohlc

df = load_ohlc("EURUSD", start="2024-01-01")
# Lit automatiquement depuis barres_au_sol si configuré
```

---

## Contribuer

Pull requests bienvenues !

**Priorités** :
- Support d'autres sources (ex: Interactive Brokers, Alpha Vantage)
- Améliorations de la gestion d'erreurs Dukascopy
- Tests unitaires

---

## Licence

MIT

---

## Liens

- [Arabesque](https://github.com/ashledombos/arabesque) — Système de trading qui utilise ces données
- [Dukascopy](https://www.dukascopy.com/swiss/english/marketwatch/historical/) — Source de données
- [CCXT](https://github.com/ccxt/ccxt) — Librairie multi-exchange
- [Apache Parquet](https://parquet.apache.org/) — Format de stockage columnar
