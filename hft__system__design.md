### Byte Monk, HFT System Design
```
High Frequency Trading Systems are engineered for Speed, not milliseconds but microseconds and even nanoseconds

Topics:-
1. How Market Data is ingested
2. How an In-Memory order book works
3. How decisions are made using FPGA's and strategy engines, and
4. How orders are routed to Exchanges like NASDAQ
```

### HFT
```
At it's core HFT is the use of Algorithms and Machines to trade financial instruments like Stocks or Options at extremely High Speeds, we are talking millions of trades per second, all happening faster than a human can blink, The goal is to make Tiny profits, some time just a fraction of a cent on each trade, but to do it on such high volume and speed, that it adds up to massive gains

1. Algorithms: Computer Programs that executes Trades automatically
2. Machines: Hardware that performs trades at high speeds
3. High Speeds: The Rapid execution of trades
4. High Volume: The large number of trades executed
5. Tiny Profits: Small gains made on each trade

These Systems look for tiny inefficiency in the market
1. Price differences between exchanges
2. Temporary imbalances in the order book
3. Slow price updates
This is to do that so we can jump in before anyone else can,
Hence Speed is everything
```


### In we go
```
The first step in an HFT pipeline is receiving Market Data, the real time feed of prices, volume and order book updates from stock exchange
But here we are not talking about your everyday API or websocket feed, HFT systems use Multicast Feed directly delivered over ultra low latency networks often inside a co-location facility(physically near the exchange server to reduce time)
This data is received through specialized hardware, an ultra low latency NIC(Network Interface Card) and a custom TCP stack, sometimes even Kernel ByPass mechanism like DPDK or solarflare onload.
This allow the system to handle Market updates in microseconds, skipping the overhead of regular network stacks

Then comes the Market Data feed handler, a critical component that passes the raw stream, decodes the protocol and transforms it into a format that the system can understand, Think of it as the translator between the exchange's language and your internal logic, but it has to translate millions of messages per second

Once the market data is ingested and decoded, the next critical step is updating the order book, that is the live snapshot of all current buy and sell orders, HFT systems maintain entire order book in memory to avoid any Disk I/O or DB latency, It's updated in real time with every incoming message triggering a precise update.
In most systems you see a replicated order book like replica A and replica B kept in sync using in memory replication, this ensures Fault Tolerance, so if one replica crashes the other will take over it

The order book isn't for record keeping, it drives the rest of the pipeline, every trading decision, every market making strategy starts with the current state of the book.
These updates are then published into an event stream ready for the other components like the trading logic, FGPA engine, or smart router to consume in near zero latency.
And as soon as the order book is updated , the new market state is published into an event driven pipeline, the Backbone of realtime processing in HFT
This pipeline is built around a lock-free queue, optimized for throughput and low contention.
Why lock free, because even the slightest delay caused by locking threads can impact trade timing.
Each event like a pricing change or a new bid is stamped using a nano precision clock. This level of time and accuracy allows the system to maintain the exact sequence of market updates, benchmark internal component latencies and most importantly sync perfectly with external systems like FPGA engines and exchanges, The result is a precise timestamp stream of market events that down stream systems like trading strategies, risk engines, or smart routers can consume in real time.

In HFT, precision is power, knowing exactly when something happened is just as important as knowing what happened.



Now we enter the most hardware optimized part of the pipeline
FGPA acceleration 
FGPA stands for Field Programmable Gate Array
It is a type of reconfigurable chip, that can run custom logic at the speed of hardware, without the overhead of CPU or OS.
In HFT, FGPA's are used for `Tick To Trade` execution, Meaning the moment a tick or a market event arrives, it's evaluated by the logic on the FGPA and a trading decision can be made in sub-microsecond latency, by the time a CPU thread spins up, the FGPA has already evaluated the opportunity and fired off an order. These FGPA's are often directly connected to the event queue, receive nanosecond timestamped events and run predefined trading strategies.
Think arbitrage, market making or code stuffing, all wired into silicon. Some firms even go a step further. They push the entire decision making logic into the FGPA, to bypass software completely, this also comes with complexity
FGPA code is written in very log or VHSDL and every logic path must be deterministic. But when done right, it gives you the fastest edge in the market
Now even though FGPA's gives us ultra low latency, most trading strategies often run on software based strategy engines
A Market making engine listens to the engine stream, evaluates the current state of the order book and makes rapid decisions, like should we go tighter, should we widen the spread or should we pull our orders
This engines can be rule based, statistical or even use light weight machine learning models, Whatever the strategy is the focus is on speed and predictability
The strategy engine is the brain of the engine, a brain that thinks in micro-seconds

Once a decision is made the order is pushed to smart auto router, which takes care of where and how to execute

Once a trading strategy decides to place an order, it's not blindly fired off to an exchange, it's first routed through a smart order router, a component that decides where and how to send the order for optimal execution, Should it go to NASDAQ, NYSE, ot should it be a market order or a limited order, The router evaluates multiple venues in real time based on liquidity, latency, fill probability and even rebate structures. But before the order goes out, it passes through pre trade risk checks, these are absolutely critical for preventing financial disasters. The risk engine ensures that you are not overspending, the order is not too big and the strategy is not misfiring due to a bug
Speed must never override safety

```