Secdb
=======

This library provides compact storage of timeseries of security market data snapshots.

It is a fork of Maxim Lapshin's https://github.com/maxlapshin/stockdb project that has
been rewriten to support a more compact format, more detailed trade logging, and
supplemented with a standalone C++ database reader.

The database implements an append-only online-compressed storage, that supports
failover, indexing and simple lookups of stored quotes.

It can compress 400 000 daily quotes into 4 MB of disk storage.

Usage
=====

First install and compile it. Include it as a rebar dependency.

It can be used either as an appender, either as reader. In current implementation
these two modes cannot be mixed.

Writing database: Appender
==========================

Typical workflow when appending data to DB:

    {ok, Appender} = secdb:open_append('NASDAQ.AAPL', "2012-01-15", [{depth, 2}]),
    Appender1 = secdb:append({md, 1326601810453, [{450.1,100},{449.56,1000}], [{452.43,20},{454.15,40}]}, Appender),
    secdb:close(Appender1).


Now lets explain, what is happening.

* Open appender. Symbol name should be a symbol, date should either erlang date `{YYYY,MM,DD}`, either a string `"YYYY-MM-DD"`.
* Don't forget to specify proper depth. If you skip it, default depth is 1 and you will save only best bid and best ask
* Specify also `{scale, 1000}` option, if you want to store quotes with precision less than 1 cent. Secdb stores your prices as int: `round(Price*Scale)`
* Now append market data.
* Market data is following: ```{md, UtcMilliseconds, [{L1BidPrice,L1BidSize},{L2BidPrice,L2BidSize}..], [{L1AskPrice,L2AskSize}..]}```
* You can include ```-include_lib("secdb/include/secdb.hrl").``` to use ```#md{}``` and ```#trade{}``` records

Now take a look at db/stock folder. There you can see new file `db/stock/NASDAQ.AAPL-2012-01-15.stock` and now you can read back stocks from it.


Reading database
================

Read the whole DB
-----------------

The most simple way is just to read all daily events to replaying them

    {ok, Events} = secdb:events('NASDAQ.AAPL', "2012-01-15").

Get candle for the whole day or specified time range:

    DayCandleEvents = secdb:events('NASDAQ.AAPL', "2012-08-10", [{filter, candle, [{period, undefined}]}]).
    RangeCandleEvents = secdb:events('NASDAQ.AAPL', "2012-08-10", [{range, {15,0,0}, {16,0,0}}, {filter, candle, [{period, undefined}]}]).

But there are possible more enhanced ways of limiting amount of loaded data.


Iterator
--------

If you need something more complex than just getting all data from DB for stock/date pair, you can use iterators.
Iterator is database opened for read-only with (optionally) filters applied on it.
You can read iterator's events one-by-one, saving memory by not keeping extracted data.

Basic iterator is created as follows:

    {ok, Iterator} = secdb:init_reader('NASDAQ.AAPL', {2012, 8, 7}, []).

Here first argument is stock, second is date, third is list of filters (empty for basic case).

You can read events one-by-one using `secdb:read_event/1` function:

    {Event1, Iterator1} = secdb:read_event(Iterator),
    {Event2, Iterator2} = secdb:read_event(Iterator1).

When there are no more events, `eof` event is returned. Make sure your code handles it well!

Also, you can call `secdb:events(Iterator)` to get all events from it.


Iterator filters
----------------

Iterator filters currently may be `{range, Start, End}` or `{filter, FilterFun, FilterState0}`.
`FilterFun` may be function name from module `secdb_filters` (currently only `candle`) or
any function with arity 2 which returns list of emitted events. Filter must accept events from
previous filter (or `#md{}` and `trade{}` from DB) and `eof` to handle end of underlying source.
Return value is tuple with list of emitted events on first place and next state on second.

For example, this simple function drops every second event:

    FilterFun = fun
        (eof, _State) -> {[], eof};
        (Event, true) -> {[Event], false};
        (_Event, _Other) -> {[], true}
      end.

And, we can see it is working:

    25> length(secdb:events(Iterator)).          
    20703
    27> {ok, FIterator} = secdb:init_reader('NASDAQ.AAPL', "2012-08-07", [{filter, FilterFun, false}]),
    27> length(secdb:events(FIterator)).
    10351

To use pre-defined filters you can just specify filter name:

    28> {ok, CIterator} = secdb:init_reader('NASDAQ.AAPL', "2012-08-07", [{filter, candle, [{period, 120000}]}]),
    28> length(secdb:events(CIterator)).
    2242

SecDB index is optimized for fast timestamp seeking, so you can use `{range, Start, End}` pseudo-filter. Start and End (if defined)
are both millisecond timestamps or erlang-style `{HH, MM, SS}` tuples (tuples will work only over DB source, not over other iterator). `undefined` for `Start` or `End` means the very beginning or the very end respectively. Example:

    31> {ok, RIterator} = secdb:init_reader('NASDAQ.AAPL', "2012-08-07", [{range, {14,0,0}, {15,0,0}}]),
    31> length(secdb:events(RIterator)).
    5139
    49> {ok, HIterator} = secdb:init_reader('NASDAQ.AAPL', "2012-08-07", [{range, undefined, 1344348900451}]),
    49> length(secdb:events(HIterator)).                                                                      
    1954

Of course, you may specify multiple filters:

    32> {ok, RFCIterator} = secdb:init_reader('NASDAQ.AAPL', "2012-08-07", [{range, {14,0,0}, {15,0,0}}, {filter, FilterFun, false}, {filter, candle, [{period, 120000}]}]),
    32> length(secdb:events(RFCIterator)).
    372                                   

Also, iterators may cascade:

    35> {ok, RIterator_F} = secdb:init_reader(RIterator, [{filter, FilterFun, false}]),
    35> {ok, RIterator_F_C} = secdb:init_reader(RIterator_F, [{filter, candle, [{period, 120000}]}]),
    35> length(secdb:events(RIterator_F_C)).
    372
    36> secdb:events(RIterator_F_C) == secdb:events(RFCIterator).
    true


Self-sufficient read-only state
-------------------------------

Function `secdb:init_reader/3` currently accesses file directly. If you have distributed setup, it will fail. Secdb is able to bypass this by using a lower-level `secdb:open_read/2`.
`open_read/2` returns in-memory read-only database state with full buffer and file descriptor closed. Actually, `secdb:init_reader/3` first opens DB using `secdb:open_read/2` and then calls `secdb:init_reader/2` on it. So does `secdb:events/2`. You can do the same:

    42> {ok, S} = secdb:open_read('NASDAQ.AAPL', {2012, 8,7}),
    42> {ok, Iterator} = secdb:init_reader(S, []),
    42> secdb:events(S) == secdb:events(Iterator).
    true

Note that we still use the same Iterator which matches perfectly. `secdb:init_reader(S, [])` can be called when original file is unavailable allowing to minimize network load when DB content is needed on other node.


Querying existing data
======================

There are simple functions which let you know what data you have.
* To list all stocks having any data in database, use `secdb:stocks()`
* To list dates when some stock has any data, use `secdb:dates(Symbol)`
* To get date intersection between multiple stocks, use `secdb:common_dates([Symbol1, Symbol2, ...])`
* To get some information about file, secdb instance or stock/date pair, use `secdb:info(Secdb)`, `secdb:info(Filename)`, `secdb:info(Symbol, Date)`, `secdb:info(Symbol, Date, [Key1, Key2, ...])`. Key can be one of `path, stock, date, version, scale, depth, interval, presence`. Return value is tuplelist. Presence is `{ChunkCount, [ChunkNumber1, ChunkNumber2, ...]}`, representing some internal report.
