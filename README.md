# pythoncode
from __future__ import (absolute_import, division, print_function,
                        unicode_literals)
import datetime  
import backtrader as bt
import backtrader.analyzers as btanalyzers
from tabulate import tabulate

def printTradeAnalysis(analyzer):
    '''
    Function to print the Technical Analysis results in a nice format.
    '''
    # Get the results we are interested in
    total_open = analyzer.total.open
    total_closed = analyzer.total.closed
    total_won = analyzer.won.total
    total_lost = analyzer.lost.total
    win_streak = analyzer.streak.won.longest
    lose_streak = analyzer.streak.lost.longest
    pnl_net = round(analyzer.pnl.net.total, 2)
    strike_rate = (total_won / total_closed) * 100
    # Designate the rows
    h1 = ['Total Open', 'Total Closed', 'Total Won', 'Total Lost']
    h2 = ['Strike Rate', 'Win Streak', 'Losing Streak', 'PnL Net']
    r1 = [total_open, total_closed, total_won, total_lost]
    r2 = [strike_rate , win_streak, lose_streak, pnl_net]
    # Check which set of headers is the longest.
    if len(h1) > len(h2):
        header_length = len(h1)
    else:
        header_length = len(h2)
    # Print the rows
    print_list = [h1, r1, h2, r2]
    row_format = "{:<15}" * (header_length + 1)
    print("Trade Analysis Results:")
    for row in print_list:
        print(row_format.format(' ', *row))


class tradelist(bt.Analyzer):

    def get_analysis(self):
        return self.trades

    def __init__(self):
        self.trades = []
        self.cumprofit = 0.0

    def notify_trade(self, trade):

        if trade.isclosed:

            brokervalue = self.strategy.broker.getvalue()

            dir = 'short'
            if trade.history[0].event.size > 0: dir = 'long'

            pricein = trade.history[len(trade.history)-1].status.price
            priceout = trade.history[len(trade.history)-1].event.price
            datein = bt.num2date(trade.history[0].status.dt)
            dateout = bt.num2date(trade.history[len(trade.history)-1].status.dt)
            if trade.data._timeframe >= bt.TimeFrame.Days:
                datein = datein.date()
                dateout = dateout.date()

            pcntchange = 100 * priceout / pricein - 100
            pnl = trade.history[len(trade.history)-1].status.pnlcomm
            pnlpcnt = 100 * pnl / brokervalue
            barlen = trade.history[len(trade.history)-1].status.barlen
            pbar = pnl / barlen
            self.cumprofit += pnl

            size = value = 0.0
            for record in trade.history:
                if abs(size) < abs(record.status.size):
                    size = record.status.size
                    value = record.status.value

            highest_in_trade = max(trade.data.high.get(ago=0, size=barlen+1))
            lowest_in_trade = min(trade.data.low.get(ago=0, size=barlen+1))
            hp = 100 * (highest_in_trade - pricein) / pricein
            lp = 100 * (lowest_in_trade - pricein) / pricein
            if dir == 'long':
                mfe = hp
                mae = lp
            if dir == 'short':
                mfe = -lp
                mae = -hp

            self.trades.append({'ref': trade.ref, 'ticker': trade.data._name, 'dir': dir,
                 'datein': datein, 'pricein': pricein, 'dateout': dateout, 'priceout': priceout,
                 'chng%': round(pcntchange, 2), 'pnl': pnl, 'pnl%': round(pnlpcnt, 2),
                 'size': size, 'value': value, 'cumpnl': self.cumprofit,
                 'nbars': barlen, 'pnl/bar': round(pbar, 2),
                 'mfe%': round(mfe, 2), 'mae%': round(mae, 2)})

class firstStrategy(bt.Strategy):
    params = dict( 
        printlog = False, 
    )

    def log(self, txt, dt=None, doprint=params['printlog'] ):
        ''' Logging function fot this strategy'''
        if self.params.printlog or doprint:
            dt = dt or self.data.datetime.datetime(0)
            #print('%s, %s' % (dt.isoformat(), txt)) # print line

    def __init__(self):
        
        self.order = None
        self.buyprice = None
        self.buycomm = None
        self.range = None
        self.price1 = None
        self.price2 = None
        self.price4cc = 0
        self.price10cc = 0
        
        self.Ope4AM = 0
        self.Ope10AM = 0
        
        self.UpVol_4_Cur = 0
        self.DnVol_4_Cur = 0
        self.TotVol_4_Cur = 0
        
        self.UpVol_10_Cur = 0
        self.DnVol_10_Cur = 0
        self.TotVol_10_Cur = 0
        
        self.Dif_4_cur = 0
        self.Dif_10_cur = 0
        
        self.TotCst_4am_cur = 0
        self.TotCst_10am_cur = 0
        
        self.CC4 = 0
        self.CC10 = 0
        
    def notify_order(self, order):
        if order.status in [order.Submitted, order.Accepted]:
            return

        if order.status in [order.Completed]:
            if order.isbuy():
                self.log(
                    'LONG EXECUTED, Price: %.5f, Cost: %.2f, Comm %.5f' %
                    (order.executed.price,
                     order.executed.value,
                     order.executed.comm))
                self.log ( 'Cash, %.2f' % self.broker.get_cash())
                self.buyprice = order.executed.price
                self.sl_long = self.buyprice *  0.9993 
                
                self.buycomm = order.executed.comm
            else:  # Sell
                self.log('SHORT EXECUTED, Price: %.5f, Cost: %.2f, Comm %.5f' %
                         (order.executed.price,
                          order.executed.value,
                          order.executed.comm))
                self.log ( 'Cash, %.2f' % self.broker.get_cash())
                self.sellprice = order.executed.price
                self.sl_short = self.sellprice * 1.0007

            self.bar_executed = len(self)

        elif order.status in [order.Canceled, order.Margin, order.Rejected]:
            self.log('Order Canceled/Margin/Rejected')

        self.order = None

    def notify_trade(self, trade):
        if not trade.isclosed:
            return

        self.log('OPERATION PROFIT, GROSS %.5f, NET %.5f' %
                (trade.pnl, trade.pnlcomm ))
      
    def next(self):
        #-------------------------------------------------------------
        #4cc values calcuted
        if self.data0.datetime.time() == datetime.time(20 , 5) :
            self.price4cc = self.data0.volume[0]
            
        if (self.data0.datetime.time() >= datetime.time(20 , 10) or self.data0.datetime.time() < datetime.time(17 , 30)) :
            self.price4cc = self.data0.volume[0] + self.price4cc

        #--------------------------------------------------------------
        #10cc values calcuted
        if self.data0.datetime.time() == datetime.time(2 , 5) :
            self.price10cc = self.data0.volume[0]
            
        if self.data0.datetime.time() >= datetime.time(2 , 10) :
            self.price10cc = self.data0.volume[0] + self.price10cc
     
        #---------------------------------------------------------------
        if self.data0.datetime.time() == datetime.time(20 , 5) :
            self.Ope4AM = self.data0.open[0]
            
        if (self.data0.datetime.time() >= datetime.time(20 , 10) or self.data0.datetime.time() < datetime.time(17 , 30)) :
            self.Dif_4_cur = self.data0.close[0] + self.Ope4AM    

        #----------------------------------------------------------------
        if self.data0.datetime.time() == datetime.time(2 , 5) :
            self.Ope10AM = self.data0.open[0]
            
        if self.data0.datetime.time() >= datetime.time(2 , 10) :
            self.Dif_10_cur = self.data0.close[0] + self.Ope10AM
        
        #-----------------------------------------------------------------
        if self.Dif_4_cur != 0 :
            self.TotCst_4am_cur  = (self.UpVol_4_Cur  - self.DnVol_4_Cur)  / self.Dif_4_cur
            
        if self.Dif_10_cur != 0 :
            self.TotCst_10am_cur  = (self.UpVol_10_Cur  - self.DnVol_10_Cur)  / self.Dif_10_cur
        #------------------------------------------------------------------
        #cc4 values calcuted
        if self.Dif_4_cur >= 0 and (self.UpVol_4_Cur  - self.DnVol_4_Cur) >= 0:
            self.CC4  = abs(self.TotCst_4am_cur)
            
        if self.Dif_4_cur >= 0 and (self.UpVol_4_Cur  - self.DnVol_4_Cur) < 0:
            self.CC4  = abs(self.TotCst_4am_cur)
            
        if self.Dif_4_cur < 0 and (self.UpVol_4_Cur  - self.DnVol_4_Cur) < 0:
            self.CC4  = -(self.TotCst_4am_cur)
            
        if self.Dif_4_cur < 0 and (self.UpVol_4_Cur  - self.DnVol_4_Cur) >= 0:
            self.CC4  = (self.TotCst_4am_cur)
        #------------------------------------------------------------------
        #cc10 values calcuted
        if self.Dif_10_cur >= 0 and (self.UpVol_10_Cur  - self.DnVol_10_Cur) >= 0:
            self.CC10  = abs(self.TotCst_10am_cur)
            
        if self.Dif_10_cur >= 0 and (self.UpVol_10_Cur  - self.DnVol_10_Cur) < 0:
            self.CC10  = abs(self.TotCst_10am_cur)
            
        if self.Dif_10_cur < 0 and (self.UpVol_10_Cur  - self.DnVol_10_Cur) < 0:
            self.CC10  = -(self.TotCst_10am_cur)
            
        if self.Dif_10_cur < 0 and (self.UpVol_10_Cur  - self.DnVol_10_Cur) >= 0:
            self.CC10  = (self.TotCst_10am_cur)
        #------------------------------------------------------------------
        
        
        if self.data0.datetime.time() == datetime.time(0 , 0) :
            self.price1 = None
            self.price2 = None

        if self.data0.datetime.time() == datetime.time(1 , 0) :
            self.price1 = self.data0.open[0]

        if self.data0.datetime.time() == datetime.time(10 , 15) :
            self.price2 = self.data0.open[0]
        
        if (self.data0.datetime.time() >= datetime.time(17 , 30) and self.data0.datetime.time() <= datetime.time(18 , 30)) :
            self.price4cc = 0
            self.price10cc = 0
            
        #print(self.data.datetime.datetime(0),self.price4cc,self.data0.volume[0]) 
        
        ##  print values for diagnostics
        txt = list()
        txt.append('{}'.format(self.data0.datetime.datetime(0)))
        txt.append('{}'.format(self.data0.open[0]))
        txt.append('{}'.format(self.data0.high[0]))
        txt.append('{}'.format(self.data0.low[0]))
        txt.append('{}'.format(self.data0.close[0]))
        # print(txt)
          
        if self.order:
            return

        if  not self.position.size  and self.price1 and self.price2 :

            if self.data0.datetime.time() == datetime.time(10 , 20) :
               
                if self.price2 < self.price1 and (self.price2 - self.price1) < -0.0020 :
                    self.order = self.buy()
                    self.log('LONG CREATE, %.5f' % self.data0.close[0])
                elif self.price2 > self.price1 and (self.price2 - self.price1) > 0.0020 :
                    self.order = self.sell()
                    self.log('SHORT CREATE, %.5f' % self.data0.close[0])

        else:
            if self.position.size > 0 :

                if (self.data0.close[0] - self.buyprice ) >= ( 0.0070 ):
                    self.order = self.close()
                    self.log('LONG TARGET MET, %.5f' % self.data0.close[0])
                    
                if (self.data0.close[0] - self.buyprice ) < ( -0.0070 ) : 
            
                    self.order = self.close()
                    self.log('LONG SL HIT, %.5f' % self.data0.close[0])
                    
            elif self.position.size  < 0 :

                if (self.sellprice - self.data0.close[0] ) >= ( 0.0070 ):
                    self.order = self.close()
                    self.log('SHORT TARGET MET, %.5f' % self.data0.close[0])

                if (self.sellprice - self.data0.close[0] ) < ( -0.0070 ) : 

                    self.order = self.close()
                    self.log('SHORT SL HIT, %.5f' % self.data0.close[0])

            #if self.data0.datetime.time() == datetime.time(23 , 50) :
                #self.order = self.close()

if __name__ == '__main__':
    cerebro = bt.Cerebro() 
   
    cerebro.addstrategy (firstStrategy, printlog = True)
    cerebro.addanalyzer(bt.analyzers.TradeAnalyzer, _name="ta")
    
    data0 = bt.feeds.GenericCSVData( 
        dataname='D:\FOREXDATE\GBPUSD1.csv', 
        dtformat= ('%m/%d/%Y %H:%M'),
        datetime=0,                              
        open=1,
        high=2,
        low=3,
        close=4,
        volume=5,
        openinterest=-1,
        timeframe=bt.TimeFrame.Minutes, 
        compression=5)                  
    
    cerebro.adddata(data0)
    cerebro.broker.setcash(100000)
    
    strategies = cerebro.run()
    firstStrategy = strategies[0]
    

    cerebro.addsizer(bt.sizers.FixedSize, stake=1000)
    cerebro.addanalyzer(tradelist, _name='trade_list')
    cerebro.addanalyzer(bt.analyzers.TradeAnalyzer, _name='ta')
    strats = cerebro.run(tradehistory=True)
    trade_list = strats[0].analyzers.trade_list.get_analysis()
    
    printTradeAnalysis(firstStrategy.analyzers.ta.get_analysis())
    
    print (tabulate(trade_list, headers="keys"))
    print('Final Portfolio Value: %.2f' % cerebro.broker.getvalue())
