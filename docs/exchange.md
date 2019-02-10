# Adding a new exchange

Perhaps the best way to understand the workings of the library is to walk through the addition of a new exchange. For this example, we'll 
add support for the exchange [Huobi](https://huobi.readme.io/docs/ws-api-reference). The exchange supports websocket data, so we'll 
add support for these endopoints.


### Adding a new Feed class
The first step is to define a new class, with the Feed class as the parent. By convention new feeds go into new modules, so the 
class definition will go in the `huobi` module within `cryptofeed`. 

```python
import logging

from cryptofeed.feed import Feed
from cryptofeed.defines import HUOBI


LOG = logging.getLogger('feedhandler')


class Huobi(Feed):
    id = HUOBI

    def __init__(self, pairs=None, channels=None, callbacks=None, **kwargs):
        super().__init__('wss://api.huobi.pro/hbus/ws', pairs=pairs, channels=channels, callbacks=callbacks, **kwargs)
        self.__reset()

    def __reset(self):
        pass
    
    async def subscribe(self, websocket):
        self.__reset()

```

We've basically just extended Feed, populated the websocket address in the parent's constructor call, and defined the `__reset` and `subscribe` methods; we may or may not need `__reset` (more on this later). `subscribe` is called every time a connection is made to the exchange - typically just when the feedhandler starts, and again if the connection is interrupted and has to be reestablished. You might notice that `HUOBI` is being imported from `defines`, so we'll need to add that as well:

```python
HUOBI = 'HUOBI'
```

Again by convention the exchange names in `defines.py` are all uppercase.

### Subscribing
Cryptofeed accepts standarized names for data channels/feeds. The `Feed` parent class will convert these to the exchange specific versions for use when subscribing. Per the exchange docs, each subscription to the various data channels must be made with a new subscription message, so for this exchange we can subscribe like so:


```python
async def subscribe(self, websocket):
        self.__reset()
        client_id = 0
        for chan in self.channels:
            for pair in self.pairs:
                await websocket.send(json.dumps(
                    {
                        "sub": "market.{}.{}".format(pair, chan),
                        "id": client_id
                    }
                ))
```

This does mean we'll need to add support for the various channel mappings in `standards.py`, add support for the pair mappings in `pairs.py` and add the exchange import to `exchanges.py`. 


* `standards.py`
    - ```python
        _feed_to_exchange_map = {
            ...
            TRADES: {
                ...
                HUOBI: 'trade.detail'
            },
        ```

* `pairs.py`
    - Per the documentation we can get a list of symbols from their REST api via `GET /v1/common/symbols`
    - ```python
      def huobi_pairs():
            r = requests.get('https://api.huobi.com/v1/common/symbols').json()
            return {'{}-{}'.format(e['base-currency'].upper(), e['quote-currency'].upper()) : '{}{}'.format(e['base-currency'], e['quote-currency']) for e in r['data']}


      _exchange_function_map = {
           ...
           HUOBI: huobi_pairs
        }
      ```
* `exchanges.py`
    - ```python
      from cryptofeed.huobi.huobi import Huobi
      ```

### Message Handler
Now that we can subscribe to trades, we can add the message handler (which is called by the feedhandler when messages are received on ther websocket). Huobi's documentation informs us that messages sent via websocket are compressed, so we'll need to make sure we uncompress them before handling them. It also informs us that we'll need to respond to pings or be disconnected. Most websocket libraries will do this automatically, but they cannot intepret a ping correctly if the messages are compressed so we'll need to handle pings automatically. We also can see from the documentation that the feed and pair are sent in the update so we'll need to parse those out to properly handle the message. 


``python
async def _trade(self, msg):
        """
        {
            'ch': 'market.btcusd.trade.detail',
            'ts': 1549773923965, 
            'tick': {
                'id': 100065340982, 
                'ts': 1549757127140,
                'data': [{'id': '10006534098224147003732', 'amount': Decimal('0.0777'), 'price': Decimal('3669.69'), 'direction': 'buy', 'ts': 1549757127140}]}}
        """
        for trade in msg['tick']['data']:
            await self.callbacks[TRADES](
                feed=self.id,
                pair=pair_exchange_to_std(msg['ch'].split('.')[1]),
                order_id=trade['id'],
                side=BUY if trade['direction'] == 'buy' else SELL,
                amount=Decimal(trade['amount']),
                price=Decimal(trade['price']),
                timestamp=trade['ts']
            )

    async def message_handler(self, msg):
        # unzip message
        msg = zlib.decompress(msg, 16+zlib.MAX_WBITS)
        msg = json.loads(msg, parse_float=Decimal)
        
        # Huobi sends a ping evert 5 seconds and will disconnect us if we do not respond to it
        if 'ping' in msg:
            await self.websocket.send(json.dumps({'pong': msg['ping']}))
            print("Ping sent")
        elif 'status' in msg and msg['status'] == 'ok':
            return
        elif 'ch' in msg:
            if 'trade' in msg['ch']:
                await self._trade(msg)
        else:
            LOG.warning("%s: Invalid message type %s", self.id, msg)
```