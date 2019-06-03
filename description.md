# PricesCC - synthetic price betting "game", though most would call this trading

Users can open long and short leveraged positions against house (CC) for assets which prices recorded on a chain by trustless oracle.

## Prices:

For example, on CFEKBET1 chain can be traded:
* BTCUSD, BTC to some other currencies pairs
* Forex pairs (USD/CURRENCY)
* crypto/BTC pairs
* some stocks / USD pairs


List of the tickers available on CFEKBET1: https://paste.ubuntu.com/p/5vyDm6kjYv/ 

To get a list of prices for assets activated on chain use call:

`./komodo-cli -ac_name=name prices depth`

where depth is an integer which showing how many data samples you want to get

For example:  

```./komodo-cli -ac_name=name prices 1
{
  "firstheight": 1661,
  "timestamps": [
    1558769510
  ],
  "pricefeeds": [
    {
      "name": "BTC_USD",
      "prices": [
        [
          7975.54670000,
          7994.27330000,
          7923.69608650
        ]
      ]
    },
...
```

this call will display prices for the single (latest) timestamp. 
As an example, this: http://159.69.45.70:8050  website just issuing prices RPC call with hard-coded depth param and representing data in form of graphs.
https://github.com/tonymorony/komodo-cctools-python/blob/master/prices_visualization_server.py 

As you noticed there are 3 numbers for each sample:
* mined (actual received) price (trace 0)
* correlated price (trace 1)
* smoothed price (trace 2)

Overall trading mechanics are similar to other leveraged trading platforms: when the loss exceeds position size - it can be liquidated, also any time user can close his position to take current not negative position equity.

One difference which important is that prices showing with 24 hours delay, so actually average is fixing as your position in 24 hours so such trading might need some patience. Position price calculating as max price in 24 hours (after pricesbet tx mined) for longs and min price in 24 hours for longs

All bets placing against CC (autonomous fund), and paying by special CC address. Also, all position liquidations (rekts) are paying to this global address. Small part of rekt paying to the wallet which broadcasted valid rekt transaction (it’s generating with similar to faucetget POW by pricesrekt call).

Example of simple “rektminer”: https://github.com/tonymorony/komodo-cctools-python/blob/master/rekt_inspector.py 

## Single ticker bets: 

For single ticker things that already activated on a chain, it is very easy to place a bet, just specify which one and the position size and leverage, for example:

`./komodo-cli -ac_name=name pricesbet 100 1 "BTC_USD, 1"`

Where 100 is position size, 1 is leverage (use minus `-` sign for negative leverage what means short), “BTC_USD, 1” is synthetic. Where BTC_USD means to place a bet on BTC_USD pair, 1 is a weight coeffitent of stack - in this simple example you can abstract from this number.

But what if you want to place a bet on pair which is not activated on a chain? 

Let’s say - you want to trade KMD to Warren Buffet’s Berkshire Hathaway stock. Actually, you can!

## Multiple tickers bets:

For combining things jl777 made a simple "language" where you push prices and operations to a synthetic stack:

`“BTCUSD, USDEUR, *, 1”`

the above would multiply BTC/USD with USD/EUR and you end up with BTC/EUR

The language supports up to 3 prices on the stack, operations *, / for multiply and div

`“BTC_USD, BTC_EUR, /, 1”` would end up with EUR/USD

there is also !, which will invert a price:

`“BTC_USD, !, 1” -> USD_BTC`

And then the three symbol operators * */, *//, * * * and ///

A div symbol / is the same as multiplying by the inverse, so this way you can mix and match any three symbols to make a synthetic one where one or more of the terms in common get factored out. pretty standard forex transformations

Lets back to our KMD to Warren Buffet’s Berkshire Hathaway idea:
    
we have pair A: `KMD_BTC`
and pair B: `BRK.A_USD`

And we want to end up with something like a `KMD_BRK.A`

Let’s use 3 tickets synthetic to achieve it:

`“KMD_BTC, BTC_USD, USD_BRK.A, **/, 1”`

What it will do is multiply `KMD_BTC` and `BTC_USD` what will result a `KMD_USD`, then KMD_USD will be divided on `USD_BRK.A` (or other words USD_BRK.A will be flipped and multiplied) what will give us desired result: KMD_BRK.A

## Multiple stacks bets:

Finally, language allow adding up an arbitrary number of these 1, 2, 3 symbol values with a weight that allows creating things that work like an index:

`A/B * B/C -> A/C`

So by allowing all possible ways to combine 1, 2 and 3 symbols into a group G, we then can create a synthetic price of: G0K0 + G1K1 + ... Gn*Kn
Which is a form that allows you to create basically anything/
However it is unsigned, can’t be negative. only the entire synthetic can be negative so with two of them S1 and S2, you can put a negative leverage on S2 and get a (S1 - S2) spread
it is basically a general purpose language based on forth that can create any synthetic position desired, but most people would just be BTCUSD +/- leverage
That is where this number after arithmetical operator matters.

For example, let’s combine previous synthetic with other one:

`“KMD_BTC, BTC_USD, USD_BRK.A, **/, 1, BTC_USD, !, 3”`

First stack will give us:  (1 / 4) * KMD_BRK.A and second pair is (3 / 4) USD_BTC 
so as result we’ll receive index: 

    `KMD_BRK.A / 4 + (3 * USD_BTC) / 4`


From all bets paying flat 0.5% fee (from bet size, not leveraged size!). On CFEKBET1 chain activated paymentsCC payment plan which can divide (airdrop) these fees between CFEKBET1 coin holders. 

For sure since house is working autonomous way it should have some risk management to not accept more risk that it can afford with goal to not be drained on long distance and maybe even operate in profit. 

## The RPC calls:

pricesbet(amount,leverage,synthetic) - opening bet

priceslist([all|open|closed])  - returning list of bets executed on chain, by default returns both opened and closed bets

mypriceslist([all|open|closed]) - returning list of bets executed on chain from executor pubkey, by default returns both opened and closed bets

pricesaddfunding bettxid amount - adding amount more funding to bettxid bet to reduce risk of liquidation 

pricesinfo bettxid - returns information about bettxid bet

pricesrekt bettxid rektheight - rekt bettxid bet (bettxid should have IsRekt: 1 flag), performing some POW

pricescashout bettxid - performing bettxid bet cashout (equity should be positive, bet should be opened and not rekted)

pricesrefillfund amount - adding funds to house

pricesgetorderbook - showing balance proportions of house orderbook pairs and also house wallet balance

GUI description:

    For simplifying of user interaction with CC, was made simple web GUI. 

Portable bundles contains precompiled compatible daemon and GUI executable: https://github.com/tonymorony/komodo-cctools-python/releases/tag/0.0.1 
Main tab:

Allows to open new position, user need to input desired bet amount (number), leverage (positive or negative number) and synthetic without “, for example BTC_USD, 1


Active positions tab:



Showing user active (not closed or rekt executed) positions. Each position in list have radio button, it’s possible to Add funding or Close selected position by relevant buttons. 

Closed position (history) tab:

Showing bets history (cashouted or rekt-performed bets)

------------------------------------

CFEKBET1 chain params:

```
./komodod -ac_name=CFEKBET1 -ac_reward=700000000 -ac_supply=77777 -ac_cbopret=7 -ac_cc=313 -ac_ccenable=226,236,237,240 -ac_blocktime=300 -ac_notarypay=250000000 -ac_snapshot=288 -ac_sapling=1 -ac_earlytxidcontract=237 -ac_algo=verushash11 -ac_prices="ETH, LTC, BNB, NEO, LRC, QTUM, OMG, ZRX, STRAT, IOTA, XVG, KMD, EOS, ZEC, DASH, XRP, STORJ, XMR, BAT, BTS, LSK, ADA, WAVES, STEEM, RVN, DCR, XEM, ICX, HOT, ENJ" -ac_stocks="AAPL,ADBE,ADSK,AKAM,AMD,AMZN,ATVI,BB,CDW,CRM,CSCO,CYBR,DBX,EA,FB,GDDY,GOOG,GRMN,GSAT,HPQ,IBM,INFY,INTC,INTU,JNPR,MSFT,MSI,MU,MXL,NATI,NCR,NFLX,NTAP,NVDA,ORCL,PANW,PYPL,QCOM,RHT,S,SHOP,SNAP,SPOT,SYMC,SYNA,T,TRIP,TWTR,TXN,VMW,VOD,VRSN,VZ,WDC,XRX,YELP,YNDX,ZEN,BRK.A" -addnode=159.69.45.70 -earlytxid=86618f7fb240550f8e0cca9cb0c955caa06a54e7f8b22ab37031f6161e830444 -daemon

Chain started on https://github.com/KMDLabs/komodo/tree/pricescomp daemon.
```
