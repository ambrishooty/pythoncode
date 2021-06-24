# pythoncode
import pandas as pd
import pylab as plt

import sys
sys.path.insert(0,'../backtestengine/')

import talib
import utils
import numpy as np
from talib.abstract import *
import inspect
from scipy import stats

import barseries
import order
import trade
import position
import backtester
import backtestreports as br

import datetime
from datetime import date
from datetime import time

import strategymanager as manager

%config InlineBackend.figure_format = 'retina'
%matplotlib inline

## DATA LOAD
bars = barseries.barSeriesFromCSV(filename='./DATA/EURUSD_15M.csv',symbol='EURUSD',precision=5)
#print(bars.all_bars)

params = {
  'period': 14,
  'qty': 100000
}

today_5_close, yesterday_22_open = 0 , 0
day_1_opn, dclose , day_2_opn, preclose, day_3_opn ,last_entry_price = 0, 0 , 0, 0, 0 , 0
trend_available = False
        
## Set custom indicators
def setIndicators(bars, params=None):
    return

## onOpen() definition
def onOpen(bars, params=None):
    return
## Essai
## onClose() definition
def onClose(bars, params=None):
    global today_5_close
    global yesterday_22_open
    global day_1_opn
    global dclose
    global day_2_opn
    global preclose
    global day_3_opn
    global last_entry_price
    global trend_available
    if bars.barid > 50:
        if bars.time().hour == 0 and bars.time().minute == 0:
            day_3_opn = day_2_opn
            preclose = dclose
            day_2_opn = day_1_opn
            day_1_opn = bars.open()
            dclose = bars.close(1)
        
          # SMALL TREND CHECK  
        if bars.time().hour == 22 and bars.time().minute == 0:
            yesterday_22_open = bars.open()
        
        if bars.time().hour == 5 and bars.time().minute == 0:
            today_5_close = bars.close()
        
        if today_5_close != 0 and yesterday_22_open != 0:
            trend_available = True
    
        ########### ENTRY
        if bars.time().hour == 5 and bars.time().minute == 45 and trend_available and position.positionSide(bars.symbol) is None:
            if today_5_close < yesterday_22_open and dclose < preclose and dclose < day_3_opn:
                order.buyNextBarOnOpen(bars,params.qty)
            
            elif today_5_close > yesterday_22_open and dclose > preclose and dclose > day_3_opn:
                order.sellNextBarOnOpen(bars,params.qty)
        ############ Get EntryPrice        
        if trade.openedTradeNb() > 0:
            last_entry_price = trade.openedTrade().entryprice
            
        ####### Target and StopLoss  
        
        if position.positionSide(bars.symbol) == trade.Side.BUY:
            order.sellLimit(bars,params.qty,last_entry_price+0.003,nbbars_expiry=0)
            order.sellStop(bars,params.qty,last_entry_price-0.003,nbbars_expiry=0)
        
        if position.positionSide(bars.symbol) == trade.Side.SELL:
            order.buyLimit(bars,params.qty,last_entry_price-0.003,nbbars_expiry=0)
            order.buyStop(bars,params.qty,last_entry_price+0.003,nbbars_expiry=0)
        
        ###### Eod Exit
        if bars.time().hour == 22 and bars.time().minute == 0:
            if position.positionSide(bars.symbol) == trade.Side.BUY:
                order.sellNextBarOnOpen(bars,params.qty)
            elif position.positionSide(bars.symbol) == trade.Side.SELL:
                order.buyNextBarOnOpen(bars,params.qty)     
                
                
## RUN BACKTEST
#backtester.runBacktest(bars,onOpen,onClose,setIndicators,params=params,max_bars=1000000,closedonly=False,name='C1',note='C1')

## PRINT PERFORMANCES REPORT
#br.printFullReport(br.getLastReport(),bars,qty=100000)


manager.publish('C1',
                onOpen,
                onClose,
                setIndicators,
                params,
                symbol='EURUSD',
                instrumenttype='FX',
                timescale='15M',
                lookback=50,
                isActive=1,
                sendMailSignal=1,
                sendMobileSignal=1,
                email='aar.frt@adss.com')
