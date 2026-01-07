```

# Investment Banking Trade Processing Requirements (RQE-Core)

## 1. Domain Types

enum Side = 'BUY' | 'SELL';
enum SecurityType = 'EQUITY' | 'BOND' | 'SWAP';
enum BlockStatus = 'OPEN' | 'READY_TO_ALLOCATE' | 'ALLOCATED' | 'BUSTED';

type Instrument = {
  instrumentId : String,
  securityType : SecurityType,
  isin         : ISIN,
  currency     : Currency,
  venue        : MIC
};

type Order = {
  orderId     : OrderId,
  accountId   : AccountId,
  instrumentId: String,
  side        : Side,
  qty         : Qty,
  trader      : String
};

type Execution = {
  execId      : ExecId,
  orderId     : OrderId,
  instrumentId: String,
  qty         : Qty,
  price       : Price,
  tradeDate   : YYYYMMDD,
  venue       : MIC
};

type BlockTrade = {
  blockId     : BlockId,
  instrumentId: String,
  side        : Side,
  tradeDate   : YYYYMMDD,
  grossQty    : Qty,
  avgPrice    : Price,
  status      : BlockStatus
};

type Allocation = {
  allocId     : AllocId,
  blockId     : BlockId,
  accountId   : AccountId,
  allocQty    : Qty,
  allocPrice  : Price
};

type SettlementInstruction = {
  settleId    : SettleId,
  allocId     : AllocId,
  accountId   : AccountId,
  isin        : ISIN,
  settleDate  : Date,
  method      : 'DVP' | 'FOP',
  cashAmount  : Money
};

## 2. Messages

message ExecutionReceived.v1 = {
  execId    : ExecId,
  orderId   : OrderId,
  qty       : Qty,
  price     : Price,
  venue     : MIC
};

message BlockReady.v1 = {
  blockId   : BlockId,
  grossQty  : Qty,
  avgPrice  : Price
};

message AllocationCreated.v1 = {
  allocId   : AllocId,
  blockId   : BlockId,
  accountId : AccountId,
  allocQty  : Qty
};

message SettlementSent.v1 = {
  settleId  : SettleId,
  allocId   : AllocId
};

## 3. Tables

Instruments  : Table[Instrument];
Orders       : Table[Order];
Executions   : Table[Execution];
Blocks       : Table[BlockTrade];
Allocations  : Table[Allocation];

## 4. Systems

system MarketKafka = kafka {
  brokers  : 'k1:9092,k2:9092';
  clientId : 'trade-processing';
};

system SettlementHttp = http {
  baseUrl : 'https://settlement.internal';
};

## 5. Channels and APIs

channel ExecutionFeed =
  topic('fix.executions') via MarketKafka;

channel TradeEvents =
  topic('trade.events') via MarketKafka;

api SettlementGateway = http via SettlementHttp {
  endpoint POST '/v1/settlements' (SettlementInstruction) -> 202;
};

## 6. Delivery Guarantees

delivery ExecutionFeed: {
  semantics : at-least-once;
  dedupeKey : execId;
};

delivery TradeEvents: {
  semantics : at-least-once;
};

delivery SettlementGateway.POST '/v1/settlements': {
  semantics        : at-least-once;
  idempotencyKey   : settleId;
};

## 7. Rule: Ingest Executions

rule IngestExecution: {
  when ExecutionFeed receives e: Execution;

  must e.qty > 0;
  must e.price > 0;

  do upsert Executions by execId = e.execId with e;

  do publish TradeEvents <- ExecutionReceived.v1{
    execId  = e.execId,
    orderId = e.orderId,
    qty     = e.qty,
    price   = e.price,
    venue   = e.venue
  };
}

## 8. Rule: Build Block Trades

rule BuildBlockTrades: {
  when ExecutionFeed receives e: Execution;

  let o = single Orders as o where o.orderId = e.orderId;

  let existing =
    single Blocks as b
    where b.instrumentId = e.instrumentId
      and b.side = o.side
      and b.tradeDate = e.tradeDate
      and b.status = 'OPEN';

  let fills =
    filter(Executions, x,
      x.instrumentId = e.instrumentId
      and x.tradeDate = e.tradeDate
    );

  let grossQty = sum(fills.qty);
  let avgPx    = sum(fills.qty * fills.price) / grossQty;

  do upsert Blocks by blockId = existing.blockId with BlockTrade{
    blockId      = existing.blockId,
    instrumentId = e.instrumentId,
    side         = o.side,
    tradeDate    = e.tradeDate,
    grossQty     = grossQty,
    avgPrice     = roundPrice(avgPx),
    status       = 'READY_TO_ALLOCATE'
  };

  do publish TradeEvents <- BlockReady.v1{
    blockId  = existing.blockId,
    grossQty = grossQty,
    avgPrice = roundPrice(avgPx)
  };
}

## 9. Rule: Allocate Block Trades

rule AllocateBlock: {
  when TradeEvents receives b: BlockReady.v1;

  let block = single Blocks as x where x.blockId = b.blockId;
  let orders = all Orders as o where o.instrumentId = block.instrumentId;

  for each o in orders {
    let allocQty = block.grossQty / count(orders);

    do create Allocation{
      allocId    = makeId(block.blockId, o.accountId),
      blockId    = block.blockId,
      accountId  = o.accountId,
      allocQty   = allocQty,
      allocPrice = block.avgPrice
    };

    do publish TradeEvents <- AllocationCreated.v1{
      allocId   = makeId(block.blockId, o.accountId),
      blockId   = block.blockId,
      accountId = o.accountId,
      allocQty  = allocQty
    };
  }

  set block.status = 'ALLOCATED';
}

## 10. Rule: Generate Settlement Instructions

rule GenerateSettlement: {
  when Allocations as a created;

  let instr = single Instruments as i where i.instrumentId = a.blockId;
  let settleDate = businessDayAdd(today(), 2);

  let cash = a.allocQty * a.allocPrice;

  let si = SettlementInstruction{
    settleId   = makeId(a.allocId),
    allocId    = a.allocId,
    accountId  = a.accountId,
    isin       = instr.isin,
    settleDate = settleDate,
    method     = 'DVP',
    cashAmount = cash
  };

  do call SettlementGateway.POST '/v1/settlements' <- si;

  do publish TradeEvents <- SettlementSent.v1{
    settleId = si.settleId,
    allocId  = a.allocId
  };
}

## 11. Control: Post-Allocation Bust

rule HandleBust: {
  when Executions as e updated;

  must e.qty > 0 else {
    let block = single Blocks as b where b.instrumentId = e.instrumentId;
    set block.status = 'BUSTED';
    stop;
  };
}


```
