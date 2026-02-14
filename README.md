# Outils Barres_au_sol

Data cache Parquet pour backtesting (Dukascopy + CCXT)

Les barres sont dans au parquet.

Objectif : constituer un data lake local en Parquet (minute 1) et dériver 5m/1h/4h pour Backtrader / vectorbt.

Principe :
- source de vérité : Parquet « min1 » ;
- dérivés recalculables à la demande ;
- téléchargements incrémentaux : relancer sans risque ;
- orchestration par fichier `instruments.csv`.

Arborescence :
data/
- dukascopy/
  - min1/
  - derived/
- ccxt/
  - min1/
  - derived/

scripts :
- data_backends.py
- data_orchestrator.py
- README.md

## Dépendances

Dans ton venv :
pip install pandas numpy pyarrow requests ccxt tqdm

Optionnel (si tu fais le smoke test Backtrader) :
pip install backtrader

Optionnel (si tu fais du criblage) :
pip install vectorbt

## Fichier instruments

Le fichier `instruments.csv` décrit :
- instrument « FTMO » ;
- backend (dukascopy ou ccxt) ;
- symbole utilisé par le backend (parfois identique, parfois à ajuster) ;
- exchange (pour ccxt) ;
- price_scale (pour dukascopy, souvent 1e5 sur le forex).

## Commandes principales

Téléchargement / mise à jour (tous instruments du csv) :
python data_orchestrator.py --start 2024-01-01 --end 2024-01-02

Téléchargement avec limitation de charge :
python data_orchestrator.py --start 2024-01-01 --end 2025-12-31 --sleep-ms 200 --sleep-between 2 --duk-workers 1 --ccxt-workers 1

Vérification sans téléchargement (plan : couverture + manque) :
python data_orchestrator.py --start 2024-01-01 --end 2025-12-31 --plan

Ne traiter qu’une source :
python data_orchestrator.py --start 2024-01-01 --end 2025-12-31 --only dukascopy
python data_orchestrator.py --start 2024-01-01 --end 2025-12-31 --only ccxt

Ne traiter qu’un sous-ensemble :
python data_orchestrator.py --start 2024-01-01 --end 2025-12-31 --filter "^(EUR|USD|GBP|JPY)"
python data_orchestrator.py --start 2024-01-01 --end 2025-12-31 --filter "^(BTC|ETH)"

Recalculer uniquement les dérivés (sans download) :
python data_orchestrator.py --start 2024-01-01 --end 2025-12-31 --derive-only

Dérivés personnalisés (ex : 5m 1h 4h) :
python data_orchestrator.py --start 2024-01-01 --end 2025-12-31 --derive 5m 1h 4h

## Mode service

Cron quotidien (exemple) :
0 3 * * * cd /chemin/projet && . .venv/bin/activate && python data_orchestrator.py --start 2024-01-01 --end 2026-12-31 --sleep-ms 200 --sleep-between 2

Systemd user (idée) :
- un service qui exécute la commande ci-dessus ;
- un timer quotidien.

## Smoke test Backtrader

Créer bt_smoke_test.py :

import pandas as pd
import backtrader as bt

class Feed(bt.feeds.PandasData):
    params = dict(datetime=None, open="open", high="high", low="low", close="close", volume="volume", openinterest=-1)

class BuyHold(bt.Strategy):
    def next(self):
        if not self.position:
            self.buy(size=1)

df = pd.read_parquet("data/dukascopy/derived/EURUSD_1h.parquet").sort_index()
if getattr(df.index, "tz", None) is not None:
    df.index = df.index.tz_convert(None)

c = bt.Cerebro()
c.broker.setcash(10_000)
c.adddata(Feed(dataname=df))
c.addstrategy(BuyHold)
c.run()

Lancer :
python bt_smoke_test.py

## Comment connaître les symboles

CCXT (liste des marchés disponibles, exemple Binance) :
python -c "import ccxt; ex=ccxt.binance(); ex.load_markets(); print(list(ex.markets.keys())[:200])"

Recherche rapide d’un symbole (ex : tout ce qui contient BTC) :
python -c "import ccxt; ex=ccxt.binance(); ex.load_markets(); print([m for m in ex.markets.keys() if 'BTC' in m][:200])"

Dukascopy :
- pas de listing officiel simple ;
- test par instrument sur 1–2 jours : si vide, symbole à ajuster ;
- le script s’arrête proprement sur une série d’échecs (fail streak), donc pas de blocage global.

Remarque : les indices/énergies « .cash » FTMO ne correspondent pas toujours 1:1 à Dukascopy ; la colonne `data_symbol` permet d’ajuster au cas par cas.

