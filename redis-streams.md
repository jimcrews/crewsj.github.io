## Start Docker

``` shell
docker run --name redis-test -p 6379:6379 -d redis
```

## Connect for Redis CLI

``` shell
docker exec -it redis-test redis-cli
```

You use the `XADD` command to add an entry to a stream:

`XADD mystream * field1 value1 field2 value2 ...`

 - `mystream`: the name of your stream.
 - `*`: tells Redis to generate an ID automatically (based on timestamp).
 - `field1 value1 ...`: key-value pairs representing the message.

XADD returns the ID of the newly added entry in the stream

### Example Movie Ticket Sales

### XADD

``` shell
XADD tickets * name "Jim Crews" seat "B12" movieId 53 sessionId 832
XADD tickets * name "Arthur Crews" seat "B13" movieId 53 sessionId 832
XADD tickets * name "Beth Harmon" seat "B11" movieId 53 sessionId 832
```

### XREAD

Just read the entire stream, or use COUNT 10 to limit how many entries to return

``` shell
XREAD STREAMS tickets 0-0
```

or Block and Read any new entries ($ means new entries only)

``` shell
XREAD BLOCK 0 STREAMS tickets $
```

XREAD is not a loop. It returns immediately once it has a result. It does not keep listening for multiple entries in a loop - unless you write a loop in your application.

``` shell
last_id = "$"

loop forever:
    entries = XREAD BLOCK 0 STREAMS tickets last_id
    for stream, messages in entries:
        for id, fields in messages:
            process(fields)
            last_id = id
```


## Consumer Groups

Consumer groups allow multiple consumers to coordinate message processing without duplication. Redis tracks message delivery per consumer.

### Create a Consumer Group

```sh
XGROUP CREATE tickets ticketgroup $ MKSTREAM
```

- `tickets`: stream name
- `ticketgroup`: group name
- `$`: read only new messages
- `MKSTREAM`: creates stream if it doesn't exist

---

### Add More Messages

```sh
XADD tickets * name "Alma Wheatley" seat "B10" movieId 53 sessionId 832
XADD tickets * name "Vasily Borgov" seat "B14" movieId 53 sessionId 832
```

---

### Read with a Consumer in a Group

```sh
XREADGROUP GROUP ticketgroup consumer-1 BLOCK 0 STREAMS tickets >
```

- `GROUP ticketgroup consumer-1`: use `consumer-1` in group `ticketgroup`
- `>`: only new, never-delivered entries
- `BLOCK 0`: wait for entries if none

---

### Acknowledge Processed Messages

```sh
XACK tickets ticketgroup <message-id>
```

---

### View Pending Messages

```sh
XPENDING tickets ticketgroup
```

---

### Claim a Stuck Message

If a consumer crashes or fails to ACK:

```sh
XCLAIM tickets ticketgroup consumer-2 60000 <message-id>
```

- `60000`: minimum idle time (ms) before claiming



## Create Python Script

``` shell
import redis

r = redis.Redis(host='localhost', port=6379)
r.set('test-key', 'hello')
print(r.get('test-key'))
```