# stream-handbook

這份文件說明 [node.js](http://nodejs.org/) [streams](http://nodejs.org/docs/latest/api/stream.html) 的基本使用方法 

其他翻譯版本

**[简体中文](https://github.com/jabez128/stream-handbook)**

[![cc-by-3.0](http://i.creativecommons.org/l/by/3.0/80x15.png)](http://creativecommons.org/licenses/by/3.0/)

# node 套件指令

使用 `npm` 安裝英文版文件，輸入下列指令

```
npm install -g stream-handbook
```

現在你可以使用 `stream-handbook` 命令來開啟在 `$PAGER` 中的這份文件並且繼續閱讀這份文件

# 簡介

```
"We should have some ways of connecting programs like garden hose--screw in
another segment when it becomes necessary to massage data in
another way. This is the way of IO also."
```

[Doug McIlroy. October 11, 1964](http://cm.bell-labs.com/who/dmr/mdmpipe.html)

![doug mcilroy](http://substack.net/images/mcilroy.png)

***

Streams 源自於
[早期的 unix](http://www.youtube.com/watch?v=tc4ROCJYbm0)
幾十年來也已經證明是個使用小型組件（[做一件事](http://www.faqs.org/docs/artu/ch01s06.html)）
建立大型系統可靠的方式。

在 unix, streams 由命令列的 `|` 管程實現。

例如看目前運行的程序中有沒有 nginx
```
ps -A | grep nginx
```

在 node, 內建的
[stream 模組](http://nodejs.org/docs/latest/api/stream.html)
被很多核心套件使用也可以被使用者使用。

和 unix 相似， node stream 模組的主要組成操作者被稱為
`.pipe()` 而且你可以自由限制寫入
客戶端的速度。

Streams 可以幫助
[分離你的顧慮](http://www.c2.com/cgi/wiki?SeparationOfConcerns)
因為他們會限制實作的表面進入可以被
[重複使用](http://www.faqs.org/docs/artu/ch01s06.html#id2877537)
的一貫介面。
你可以將一個回傳的 stream 帶入另一個抽象操作 streams 來進行流程控制的
[套件](http://npmjs.org)或程式庫。

Streams 是
[small-program design](https://michaelochurch.wordpress.com/2012/08/15/what-is-spaghetti-code/)
和 [unix philosophy](http://www.faqs.org/docs/artu/ch01s06.html)
很重要的一部份
但是還有更多抽象的部分值得考慮。

一定要記得[技術債](http://c2.com/cgi/wiki?TechnicalDebt)
是敵人也要尋找解決問題的最佳抽象。

![brian kernighan](http://substack.net/images/kernighan.png)

***

# 為甚麼應該使用 streams

I/O 在 node 裡可以使用非同步的方法操作，所以你可以使用 callbacks 來操作網路和檔案。 如果你想用個檔案伺服，你可能會用這種方式：

``` js
var http = require('http');
var fs = require('fs');

var server = http.createServer(function (req, res) {
    fs.readFile(__dirname + '/data.txt', function (err, data) {
        res.end(data);
    });
});
server.listen(8000);
```

這段程式碼可以運作，可是程式在回應每個請求前先將整個檔案 `data.txt` 讀進記憶體中。
如果 `data.txt` 這個檔案非常大，這個程式如果有很多使用者同時連線的話會佔用很多的記憶體，尤其是連線速度慢的使用者。

因為使用者必須等待整個檔案緩存在伺服器的記憶體中才會開始下載內容
所以使用者的經驗也會非常的不佳。

幸好 `(req, res)` 兩個參數都是 streams，這表示我們可以使用 `fs.createReadStream()` 代替
`fs.readFile()`：

``` js
var http = require('http');
var fs = require('fs');

var server = http.createServer(function (req, res) {
    var stream = fs.createReadStream(__dirname + '/data.txt');
    stream.pipe(res);
});
server.listen(8000);
```

`.pipe()` 從 `fs.createReadStream()` 接管了監聽 `'data'` 和 `'end'` 事件。
這段程式碼不僅簡潔也讓 `data.txt` 檔案的區塊被讀取時會立即寫入客戶端。

使用 `.pipe()` 還有其他優點，例如自動處理讀取壓力
使得 node 當客戶端連線非常緩慢或是高延遲的連線時
不需要將資料區塊緩存到記憶體。

想要壓縮檔案嗎? 這當然也有 streaming 模組！

``` js
var http = require('http');
var fs = require('fs');
var oppressor = require('oppressor');

var server = http.createServer(function (req, res) {
    var stream = fs.createReadStream(__dirname + '/data.txt');
    stream.pipe(oppressor(req)).pipe(res);
});
server.listen(8000);
```

現在我們的檔案被壓縮給支援 gzip 或是 deflate 的瀏覽器！ 我們可以
讓 [oppressor](https://github.com/substack/oppressor) 處理所有
content-encoding 的事情。

一旦學習了 stream api，你就可以像樂高或是軟管將 streams 模組結合一起
而不用去記憶如何在沒有串流特定的 APIs 中使用資料。

Streams 使在 node 中編寫程式更加簡單、優雅且可分割。

# 基本

有五種基本的 streams：readable、writable、transform、duplex 和 classic。

## pipe

所有的 streams 都是使用 `.pipe()` 來配對輸入和輸出。

`.pipe()` 是個將可讀取的 stream `src` 綁定輸出到可寫入的 stream `dst` 函數：

```
src.pipe(dst)
```

`.pipe(dst)` 回傳 `dst` 所以你也可以一起鏈接更多 `.pipe()` 函數：

``` js
a.pipe(b).pipe(c).pipe(d)
```
等同於：
``` js
a.pipe(b);
b.pipe(c);
c.pipe(d);
```

上面的程式碼就等同於在命令列輸入：

```
a | b | c | d
```

## readable streams

可讀取的 streams 藉由呼叫 `.pipe()` 產生可被 writable, transform, duplex stream 使用的資料：

``` js
readableStream.pipe(dst)
```

### 建立一個 readable stream

讓我們來建立一個 readable stream!

``` js
var Readable = require('stream').Readable;

var rs = new Readable;
rs.push('beep ');
rs.push('boop\n');
rs.push(null);

rs.pipe(process.stdout);
```

```
$ node read0.js
beep boop
```

`rs.push(null)` 告訴消費者 `rs` 已經可以開始輸出資料了。

注意：
這邊我們是先將資料存進 `rs` 才連接
`process.stdout`，但是所有的資料還是會被寫入。

這是因為 `.push()` 函數會將參數的資料緩存直到消費者開始讀取。

然而，在很多情況下更好的方法是，我們避免緩存所有資料並且只在消費者要求時產生資料。

我們可以定義 `._read` 函數來存入要求的區塊資料：

``` js
var Readable = require('stream').Readable;
var rs = Readable();

var c = 97;
rs._read = function () {
    rs.push(String.fromCharCode(c++));
    if (c > 'z'.charCodeAt(0)) rs.push(null);
};

rs.pipe(process.stdout);
```

```
$ node read1.js
abcdefghijklmnopqrstuvwxyz
```

這段程式碼我們只在使用者讀取 stream 的時候存入 `'a'` 到 `'z'`。

`_read` 函數也可以提供 `size` 參數來設定消費者要讀取的位元數，不過 readable stream 也可以忽略 `size` 的設定。

你也可以使用 `util.inherits()` 來繼承 Readable stream，但是這種方法並不容易被理解。

我們將我們的程式碼加入了一些延遲以讓各位瞭解 `_read` 只會在消費者要求資料時被呼叫：

```js
var Readable = require('stream').Readable;
var rs = Readable();

var c = 97 - 1;

rs._read = function () {
    if (c >= 'z'.charCodeAt(0)) return rs.push(null);
    
    setTimeout(function () {
        rs.push(String.fromCharCode(++c));
    }, 100);
};

rs.pipe(process.stdout);

process.on('exit', function () {
    console.error('\n_read() called ' + (c - 97) + ' times');
});
process.stdout.on('error', process.exit);
```

在這個程式中 `_read()` 只在我們請求5 bytes 的資料時被呼叫了5次：

```
$ node read2.js | head -c5
abcde
_read() called 5 times
```

因為作業系統需要一些時間告訴我們關閉 pipe 的相關信號，所以必須設定 setTimeout 的延遲。

因為當 `read` 不再輸出就會對 `process.stdout` 發送 EPIPE 的錯誤，這時作業系統會對我們的程式發送 SIGPIPE 信號，所以必須要監聽 `process.stdout.on('error', fn)`。

當我們用 node 直接與外部作業系統的管程連接時，就必須加上這些額外複雜的程式。

如果想建立 readable stream 儲存非字串抽象的資料，記得建立 readable stream 時帶入 `Readable({ objectMode: true })`.

### consuming a readable stream

Most of the time it's much easier to just pipe a readable stream into another
kind of stream or a stream created with a module like
[through](https://npmjs.org/package/through)
or [concat-stream](https://npmjs.org/package/concat-stream),
but occasionally it might be useful to consume a readable stream directly.

``` js
process.stdin.on('readable', function () {
    var buf = process.stdin.read();
    console.dir(buf);
});
```

```
$ (echo abc; sleep 1; echo def; sleep 1; echo ghi) | node consume0.js 
<Buffer 61 62 63 0a>
<Buffer 64 65 66 0a>
<Buffer 67 68 69 0a>
null
```

當有資料時會觸發 `'readable'` 事件接著你可以呼叫 `.read()` 從緩存裡獲得資料。

當 stream 結束且沒有資料可以讀取時 `.read()` 函數會回傳 `null`。

你也可以在呼叫 `.read(n)` 函數帶入參數 `n`，他會回傳 `n` 位元的資料。 讀取一些位元僅僅只是建議而且對物件 streams 沒有效，不過多數的 streams 都有支援。

這個範例使用 `.read(n)` 來讀取 3-byte 的塊資料:

``` js
process.stdin.on('readable', function () {
    var buf = process.stdin.read(3);
    console.dir(buf);
});
```

都是不完全的資料！

```
$ (echo abc; sleep 1; echo def; sleep 1; echo ghi) | node consume1.js 
<Buffer 61 62 63>
<Buffer 0a 64 65>
<Buffer 66 0a 67>
```

因為在內部緩存中還有其他的資料，而我們必須將剩下的資料取出來，`.read(0)` 將可以做到:

``` js
process.stdin.on('readable', function () {
    var buf = process.stdin.read(3);
    console.dir(buf);
    process.stdin.read(0);
});
```

現在已經正常讀取 3-byte 的塊資料!

``` js
$ (echo abc; sleep 1; echo def; sleep 1; echo ghi) | node consume2.js 
<Buffer 61 62 63>
<Buffer 0a 64 65>
<Buffer 66 0a 67>
<Buffer 68 69 0a>
```

當 `.read()` 給你太多資料時，你也可以使用 `.unshift()` 放回資料以便觸發讀取的邏輯。

`.unshift()` 讓我們不用複製非必要的緩存。這個範例我們建立一個可讀的解析器來拆分換行:

``` js
var offset = 0;

process.stdin.on('readable', function () {
    var buf = process.stdin.read();
    if (!buf) return;
    for (; offset < buf.length; offset++) {
        if (buf[offset] === 0x0a) {
            console.dir(buf.slice(0, offset).toString());
            buf = buf.slice(offset + 1);
            offset = 0;
            process.stdin.unshift(buf);
            return;
        }
    }
    process.stdin.unshift(buf);
});
```

```
$ tail -n +50000 /usr/share/dict/american-english | head -n10 | node lines.js 
'hearties'
'heartiest'
'heartily'
'heartiness'
'heartiness\'s'
'heartland'
'heartland\'s'
'heartlands'
'heartless'
'heartlessly'
```

線上有其他模組可以使用，這樣就不用自行開發，例如
[split](https://npmjs.org/package/split)

## writable streams

可寫入的 stream 讓你可以使用 `.pipe()` 函數將資料串接:

``` js
src.pipe(writableStream)
```

### 建立一個 writable stream

只需要定義 `._write(chunk, enc, next)` 函數就可以了:

``` js
var Writable = require('stream').Writable;
var ws = Writable();
ws._write = function (chunk, enc, next) {
    console.dir(chunk);
    next();
};

process.stdin.pipe(ws);
```

```
$ (echo beep; sleep 1; echo boop) | node write0.js 
<Buffer 62 65 65 70 0a>
<Buffer 62 6f 6f 70 0a>
```

第一個參數 `chunk` 是由生產者所寫數的資料。

第二個參數 `enc` 是由字串表示的編碼格式只有當
`opts.decodeString` 為 `false` 以及寫入的資料是字串時。

第三個參數 `next(err)` 是使用者可以寫入更多資料的 callback。
當 stream 觸發錯誤事件時，你也可以選擇性地帶入 `err` 物件。

如果寫入的資料是字串，他將會被轉成 `Buffer` 型態除非帶入參數 `Writable({ decodeStrings: false })`。

如果寫入的資料是物件，記得帶入參數 `Writable({ objectMode: true })`。

### 寫入 writable stream

為了寫入 writable stream，需要呼叫 `.write(data)` 並帶入要寫入的 `data`！

``` js
process.stdout.write('beep boop\n');
```

為了停止寫入需要呼叫 `.end()` 函數，你也可以呼叫 `.end(data)` 並帶入最後要寫入的 `data`：

``` js
var fs = require('fs');
var ws = fs.createWriteStream('message.txt');

ws.write('beep ');

setTimeout(function () {
    ws.end('boop\n');
}, 1000);
```

```
$ node writing1.js 
$ cat message.txt
beep boop
```

如果在意高流量以及緩存，當在即將寫入緩存的資料多於建立 `Writable()` 時設定的 `opts.highWaterMark` 時， `.write()` 會回傳 `false`

如果要等待緩存清空的時候，可以監聽 `'drain'` 事件。

## transform

Transform streams are a certain type of duplex stream (both readable and writable).
The distinction is that in Transform streams, the output is in some way calculated
from the input. 

You might also hear transform streams referred to as "through streams".

Through streams are simple readable/writable filters that transform input and
produce output.

## duplex

Duplex streams are 可讀取/可寫入 and both ends of the stream engage
in a two-way interaction, sending back and forth messages like a telephone. An
rpc exchange is a good example of a duplex stream. Any time you see something
like:

``` js
a.pipe(b).pipe(a)
```

you're probably dealing with a duplex stream.

## classic streams

Classic streams are the old interface that first appeared in node 0.4.
You will probably encounter this style of stream for a long time so it's good to
know how they work.

Whenever a stream has a `"data"` listener registered, it switches into
`"classic"` mode and behaves according to the old API.

### classic readable streams

Classic readable streams are just event emitters that emit `"data"` events when
they have data for their consumers and emit `"end"` events when they are done
producing data for their consumers.

`.pipe()` checks whether a classic stream is readable by checking the truthiness
of `stream.readable`.

Here is a super simple readable stream that prints `A` through `J`, inclusive:

``` js
var Stream = require('stream');
var stream = new Stream;
stream.readable = true;

var c = 64;
var iv = setInterval(function () {
    if (++c >= 75) {
        clearInterval(iv);
        stream.emit('end');
    }
    else stream.emit('data', String.fromCharCode(c));
}, 100);

stream.pipe(process.stdout);
```

```
$ node classic0.js
ABCDEFGHIJ
```

To read from a classic readable stream, you register `"data"` and `"end"`
listeners. Here's an example reading from `process.stdin` using the old readable
stream style:

``` js
process.stdin.on('data', function (buf) {
    console.log(buf);
});
process.stdin.on('end', function () {
    console.log('__END__');
});
```

```
$ (echo beep; sleep 1; echo boop) | node classic1.js 
<Buffer 62 65 65 70 0a>
<Buffer 62 6f 6f 70 0a>
__END__
```

Note that whenever you register a `"data"` listener, you put the stream into
compatability mode so you lose the benefits of the new streams2 api.

You should pretty much never register `"data"` and `"end"` handlers yourself
anymore. If you need to interact with legacy streams, use libraries that you can
`.pipe()` to instead where possible.

For example, you can use [through](https://npmjs.org/package/through)
to avoid setting up explicit `"data"` and `"end"` listeners:

``` js
var through = require('through');
process.stdin.pipe(through(write, end));

function write (buf) {
    console.log(buf);
}
function end () {
    console.log('__END__');
}
```

```
$ (echo beep; sleep 1; echo boop) | node through.js 
<Buffer 62 65 65 70 0a>
<Buffer 62 6f 6f 70 0a>
__END__
```

or use [concat-stream](https://npmjs.org/package/concat-stream) to buffer up an
entire stream's contents:

``` js
var concat = require('concat-stream');
process.stdin.pipe(concat(function (body) {
    console.log(JSON.parse(body));
}));
```

```
$ echo '{"beep":"boop"}' | node concat.js 
{ beep: 'boop' }
```

Classic readable streams have `.pause()` and `.resume()` logic for provisionally
pausing a stream, but this was merely advisory. If you are going to use
`.pause()` and `.resume()` with classic readable streams, you should use
[through](https://npmjs.org/package/through) to handle buffering instead of
writing that yourself.

### classic writable streams

Classic writable streams are very simple. Just define `.write(buf)`, `.end(buf)`
and `.destroy()`.

`.end(buf)` may or may not get a `buf`, but node people will expect `stream.end(buf)`
to mean `stream.write(buf); stream.end()` and you shouldn't violate their
expectations.

## read more

* [core stream documentation](http://nodejs.org/docs/latest/api/stream.html#stream_stream)
* You can use the [readable-stream](https://npmjs.org/package/readable-stream)
module to make your streams2 code compliant with node 0.8 and below. Just
`require('readable-stream')` instead of `require('stream')` after you
`npm install readable-stream`.

***

# 內建的 streams

下面的 streams 是在 node 中內建的.

## process

### [process.stdin](http://nodejs.org/docs/latest/api/process.html#process_process_stdin)

This readable stream contains the standard system input stream for your program.

It is paused by default but the first time you refer to it `.resume()` will be
called implicitly on the
[next tick](http://nodejs.org/docs/latest/api/process.html#process_process_nexttick_callback).

If process.stdin is a tty (check with
[`tty.isatty()`](http://nodejs.org/docs/latest/api/tty.html#tty_tty_isatty_fd))
then input events will be line-buffered. You can turn off line-buffering by
calling `process.stdin.setRawMode(true)` BUT the default handlers for key
combinations such as `^C` and `^D` will be removed.

### [process.stdout](http://nodejs.org/api/process.html#process_process_stdout)

This writable stream contains the standard system output stream for your program.

`write` to it if you want to send data to stdout

### [process.stderr](http://nodejs.org/api/process.html#process_process_stderr)

This writable stream contains the standard system error stream for your program.

`write` to it if you want to send data to stderr

## child_process.spawn()

## fs

### fs.createReadStream()

### fs.createWriteStream()

## net

### [net.connect()](http://nodejs.org/docs/latest/api/net.html#net_net_connect_options_connectionlistener)

This function returns a [duplex stream] that connects over tcp to a remote
host.

You can start writing to the stream right away and the writes will be buffered
until the `'connect'` event fires.

### net.createServer()

## http

### http.request()

### http.createServer()

## zlib

### zlib.createGzip()

### zlib.createGunzip()

### zlib.createDeflate()

### zlib.createInflate()

***

# control streams

## [through](https://github.com/dominictarr/through)

## [from](https://github.com/dominictarr/from)

## [pause-stream](https://github.com/dominictarr/pause-stream)

## [concat-stream](https://github.com/maxogden/node-concat-stream)

concat-stream will buffer up stream contents into a single buffer.
`concat(cb)` just takes a single callback `cb(body)` with the buffered
`body` when the stream has finished.

For example, in this program, the concat callback fires with the body string
`"beep boop"` once `cs.end()` is called.
The program takes the body and upper-cases it, printing `BEEP BOOP.`

``` js
var concat = require('concat-stream');

var cs = concat(function (body) {
    console.log(body.toUpperCase());
});
cs.write('beep ');
cs.write('boop.');
cs.end();
```

```
$ node concat.js
BEEP BOOP.
```

Here's an example usage of concat-stream that will parse incoming url-encoded
form data and reply with a stringified JSON version of the form parameters:

``` js
var http = require('http');
var qs = require('querystring');
var concat = require('concat-stream');

var server = http.createServer(function (req, res) {
    req.pipe(concat(function (body) {
        var params = qs.parse(body.toString());
        res.end(JSON.stringify(params) + '\n');
    }));
});
server.listen(5005);
```

```
$ curl -X POST -d 'beep=boop&dinosaur=trex' http://localhost:5005
{"beep":"boop","dinosaur":"trex"}
```

## [duplex](https://github.com/dominictarr/duplex)

## [duplexer](https://github.com/Raynos/duplexer)

## [emit-stream](https://github.com/substack/emit-stream)

## [invert-stream](https://github.com/dominictarr/invert-stream)

## [map-stream](https://github.com/dominictarr/map-stream)

## [remote-events](https://github.com/dominictarr/remote-events)

## [buffer-stream](https://github.com/Raynos/buffer-stream)

## [event-stream](https://github.com/dominictarr/event-stream)

## [auth-stream](https://github.com/Raynos/auth-stream)

***

# meta streams

## [mux-demux](https://github.com/dominictarr/mux-demux)

## [stream-router](https://github.com/Raynos/stream-router)

## [multi-channel-mdm](https://github.com/Raynos/multi-channel-mdm)

***

# state streams

## [crdt](https://github.com/dominictarr/crdt)

## [delta-stream](https://github.com/Raynos/delta-stream)

## [scuttlebutt](https://github.com/dominictarr/scuttlebutt)

[scuttlebutt](https://github.com/dominictarr/scuttlebutt) can be used for
peer-to-peer state synchronization with a mesh topology where nodes might only
be connected through intermediaries and there is no node with an authoritative
version of all the data.

The kind of distributed peer-to-peer network that
[scuttlebutt](https://github.com/dominictarr/scuttlebutt) provides is especially
useful when nodes on different sides of network barriers need to share and
update the same state. An example of this kind of network might be browser
clients that send messages through an http server to each other and backend
processes that the browsers can't directly connect to. Another use-case might be
systems that span internal networks since IPv4 addresses are scarce.

[scuttlebutt](https://github.com/dominictarr/scuttlebutt) uses a
[gossip protocol](https://en.wikipedia.org/wiki/Gossip_protocol)
to pass messages between connected nodes so that state across all the nodes will
[eventually converge](https://en.wikipedia.org/wiki/Eventual_consistency)
on the same value everywhere.

Using the `scuttlebutt/model` interface, we can create some nodes and pipe them
to each other to create whichever sort of network we want:

``` js
var Model = require('scuttlebutt/model');
var am = new Model;
var as = am.createStream();

var bm = new Model;
var bs = bm.createStream();

var cm = new Model;
var cs = cm.createStream();

var dm = new Model;
var ds = dm.createStream();

var em = new Model;
var es = em.createStream();

as.pipe(bs).pipe(as);
bs.pipe(cs).pipe(bs);
bs.pipe(ds).pipe(bs);
ds.pipe(es).pipe(ds);

em.on('update', function (key, value, source) {
    console.log(key + ' => ' + value + ' from ' + source);
});

am.set('x', 555);
```

The network we've created is an undirected graph that looks like:

```
a <-> b <-> c
      ^
      |
      v
      d <-> e
```

Note that nodes `a` and `e` aren't directly connected, but when we run this
script:

```
$ node model.js
x => 555 from 1347857300518
```

the value that node `a` set finds its way to node `e` by way of nodes `b` and
`d`. Here all the nodes are in the same process but because
[scuttlebutt](https://github.com/dominictarr/scuttlebutt) uses a
simple streaming interface, the nodes can be placed on any process or server and
connected with any streaming transport that can handle string data.

Next we can make a more realistic example that connects over the network and
increments a counter variable.

Here's the server which will set the initial `count` value to 0 and `count ++`
every 320 milliseconds, printing all updates to count:

``` js
var Model = require('scuttlebutt/model');
var net = require('net');

var m = new Model;
m.set('count', '0');
m.on('update', function (key, value) {
    console.log(key + ' = ' + m.get('count'));
});

var server = net.createServer(function (stream) {
    stream.pipe(m.createStream()).pipe(stream);
});
server.listen(8888);

setInterval(function () {
    m.set('count', Number(m.get('count')) + 1);
}, 320);
```

Now we can make a client that connects to this server, updates the count on an
interval, and prints all the updates it receives:

``` js
var Model = require('scuttlebutt/model');
var net = require('net');

var m = new Model;
var s = m.createStream();

s.pipe(net.connect(8888, 'localhost')).pipe(s);

m.on('update', function cb (key) {
    // wait until we've gotten at least one count value from the network
    if (key !== 'count') return;
    m.removeListener('update', cb);
    
    setInterval(function () {
        m.set('count', Number(m.get('count')) + 1);
    }, 100);
});

m.on('update', function (key, value) {
    console.log(key + ' = ' + value);
});
```

The client is slightly trickier since it should wait until it has an update from
somebody else to start updating the counter itself or else its counter would be
zeroed.

Once we get the server and some clients running we should see a sequence like this:

```
count = 183
count = 184
count = 185
count = 186
count = 187
count = 188
count = 189
```

Occasionally on some of the nodes we might see a sequence with repeated values like:

```
count = 147
count = 148
count = 149
count = 149
count = 150
count = 151
```

These values are due to
[scuttlebutt's](https://github.com/dominictarr/scuttlebutt)
history-based conflict resolution algorithm which is hard at work ensuring that the state of the system across all nodes is eventually consistent.

Note that the server in this example is just another node with the same
privledges as the clients connected to it. The terms "client" and "server" here
don't affect how the state synchronization proceeds, just who initiates the
connection. Protocols with this property are often called symmetric protocols.
See [dnode](https://github.com/substack/dnode) for another example of a
symmetric protocol.

## [append-only](http://github.com/Raynos/append-only)

***

# http streams

## [request](https://github.com/mikeal/request)

## [oppressor](https://github.com/substack/oppressor)

## [response-stream](https://github.com/substack/response-stream)

***

# io streams

## [reconnect](https://github.com/dominictarr/reconnect)

## [kv](https://github.com/dominictarr/kv)

## [discovery-network](https://github.com/Raynos/discovery-network)

***

# parser streams

## [tar](https://github.com/creationix/node-tar)

## [trumpet](https://github.com/substack/node-trumpet)

## [JSONStream](https://github.com/dominictarr/JSONStream)

Use this module to parse and stringify json data from streams.

If you need to pass a large json collection through a slow connection or you
have a json object that will populate slowly this module will let you parse data
incrementally as it arrives.

## [json-scrape](https://github.com/substack/json-scrape)

## [stream-serializer](https://github.com/dominictarr/stream-serializer)

***

# browser streams

## [shoe](https://github.com/substack/shoe)

## [domnode](https://github.com/maxogden/domnode)

## [sorta](https://github.com/substack/sorta)

## [graph-stream](https://github.com/substack/graph-stream)

## [arrow-keys](https://github.com/Raynos/arrow-keys)

## [attribute](https://github.com/Raynos/attribute)

## [data-bind](https://github.com/travis4all/data-bind)

***

# html streams

## [hyperstream](https://github.com/substack/hyperstream)


# audio streams

## [baudio](https://github.com/substack/baudio)

# rpc streams

## [dnode](https://github.com/substack/dnode)

[dnode](https://github.com/substack/dnode) lets you call remote functions
through any kind of stream.

Here's a basic dnode server:

``` js
var dnode = require('dnode');
var net = require('net');

var server = net.createServer(function (c) {
    var d = dnode({
        transform : function (s, cb) {
            cb(s.replace(/[aeiou]{2,}/, 'oo').toUpperCase())
        }
    });
    c.pipe(d).pipe(c);
});

server.listen(5004);
```

then you can hack up a simple client that calls the server's `.transform()`
function:

``` js
var dnode = require('dnode');
var net = require('net');

var d = dnode();
d.on('remote', function (remote) {
    remote.transform('beep', function (s) {
        console.log('beep => ' + s);
        d.end();
    });
});

var c = net.connect(5004);
c.pipe(d).pipe(c);
```

Fire up the server, then when you run the client you should see:

```
$ node client.js
beep => BOOP
```

The client sent `'beep'` to the server's `transform()` function and the server
called the client's callback with the result, neat!

The streaming interface that dnode provides here is a duplex stream since both
the client and server are piped to each other (`c.pipe(d).pipe(c)`) with
requests and responses coming from both sides.

The craziness of dnode begins when you start to pass function arguments to
stubbed callbacks. Here's an updated version of the previous server with a
multi-stage callback passing dance:

``` js
var dnode = require('dnode');
var net = require('net');

var server = net.createServer(function (c) {
    var d = dnode({
        transform : function (s, cb) {
            cb(function (n, fn) {
                var oo = Array(n+1).join('o');
                fn(s.replace(/[aeiou]{2,}/, oo).toUpperCase());
            });
        }
    });
    c.pipe(d).pipe(c);
});

server.listen(5004);
```

Here's the updated client:

``` js
var dnode = require('dnode');
var net = require('net');

var d = dnode();
d.on('remote', function (remote) {
    remote.transform('beep', function (cb) {
        cb(10, function (s) {
            console.log('beep:10 => ' + s);
            d.end();
        });
    });
});

var c = net.connect(5004);
c.pipe(d).pipe(c);
```

After we spin up the server, when we run the client now we get:

```
$ node client.js
beep:10 => BOOOOOOOOOOP
```

It just works!™

The basic idea is that you just put functions in objects and you call them from
the other side of a stream and the functions will be stubbed out on the other
end to do a round-trip back to the side that had the original function in the
first place. The best thing is that when you pass functions to a stubbed
function as arguments, those functions get stubbed out on the *other* side!

This approach of stubbing function arguments recursively shall henceforth be
known as the "turtles all the way down" gambit. The return values of any of your
functions will be ignored and only enumerable properties on objects will be
sent, json-style.

It's turtles all the way down!

![turtles all the way](http://substack.net/images/all_the_way_down.png)

Since dnode works in node or on the browser over any stream it's easy to call
functions defined anywhere and especially useful when paired up with
[mux-demux](https://github.com/dominictarr/mux-demux) to multiplex an rpc stream
for control alongside some bulk data streams.

## [rpc-stream](https://github.com/dominictarr/rpc-stream)

***

# test streams

## [tap](https://github.com/isaacs/node-tap)

## [stream-spec](https://github.com/dominictarr/stream-spec)

***

# power combos

## distributed partition-tolerant chat

The [append-only](http://github.com/Raynos/append-only) module can give us a
convenient append-only array on top of
[scuttlebutt](https://github.com/dominictarr/scuttlebutt)
which makes it really easy to write an eventually-consistent, distributed chat
that can replicate with other nodes and survive network partitions.

TODO: the rest

## roll your own socket.io

We can build a socket.io-style event emitter api over streams using some of the
libraries mentioned earlier in this document.

First  we can use [shoe](http://github.com/substack/shoe)
to create a new websocket handler server-side and
[emit-stream](https://github.com/substack/emit-stream)
to turn an event emitter into a stream that emits objects.
The object stream can then be fed into
[JSONStream](https://github.com/dominictarr/JSONStream)
to serialize the objects and from there the serialized stream can be piped into
the remote browser.

``` js
var EventEmitter = require('events').EventEmitter;
var shoe = require('shoe');
var emitStream = require('emit-stream');
var JSONStream = require('JSONStream');

var sock = shoe(function (stream) {
    var ev = new EventEmitter;
    emitStream(ev)
        .pipe(JSONStream.stringify())
        .pipe(stream)
    ;
    ...
});
```

Inside the shoe callback we can emit events to the `ev` function.  Here we'll
just emit different kinds of events on intervals:

``` js
var intervals = [];

intervals.push(setInterval(function () {
    ev.emit('upper', 'abc');
}, 500));

intervals.push(setInterval(function () {
    ev.emit('lower', 'def');
}, 300));

stream.on('end', function () {
    intervals.forEach(clearInterval);
});
```

Finally the shoe instance just needs to be bound to an http server:

``` js
var http = require('http');
var server = http.createServer(require('ecstatic')(__dirname));
server.listen(8080);

sock.install(server, '/sock');
```

Meanwhile on the browser side of things just parse the json shoe stream and pass
the resulting object stream to `eventStream()`. `eventStream()` just returns an
event emitter that emits the server-side events:

``` js
var shoe = require('shoe');
var emitStream = require('emit-stream');
var JSONStream = require('JSONStream');

var parser = JSONStream.parse([true]);
var stream = parser.pipe(shoe('/sock')).pipe(parser);
var ev = emitStream(stream);

ev.on('lower', function (msg) {
    var div = document.createElement('div');
    div.textContent = msg.toLowerCase();
    document.body.appendChild(div);
});

ev.on('upper', function (msg) {
    var div = document.createElement('div');
    div.textContent = msg.toUpperCase();
    document.body.appendChild(div);
});
```

Use [browserify](https://github.com/substack/node-browserify) to build this
browser source code so that you can `require()` all these nifty modules
browser-side:

```
$ browserify main.js -o bundle.js
```

Then drop a `<script src="/bundle.js"></script>` into some html and open it up
in a browser to see server-side events streamed through to the browser side of
things.

With this streaming approach you can rely more on tiny reusable components that
only need to know how to talk streams. Instead of routing messages through a
global event system socket.io-style, you can focus more on breaking up your
application into tinier units of functionality that can do exactly one thing
well.

For instance you can trivially swap out JSONStream in this example for
[stream-serializer](https://github.com/dominictarr/stream-serializer)
to get a different take on serialization with a different set of tradeoffs.
You could bolt layers over top of shoe to handle
[reconnections](https://github.com/dominictarr/reconnect) or heartbeats
using simple streaming interfaces.
You could even add a stream into the chain to use namespaced events with
[eventemitter2](https://npmjs.org/package/eventemitter2) instead of the
EventEmitter in core.

If you want some different streams that act in different ways it would likewise
be pretty simple to run the shoe stream in this example through mux-demux to
create separate channels for each different kind of stream that you need.

As the requirements of your system evolve over time, you can swap out each of
these streaming pieces as necessary without as many of the all-or-nothing risks
that more opinionated framework approaches necessarily entail.

## html streams for the browser and the server

We can use some streaming modules to reuse the same html rendering logic for the
client and the server! This approach is indexable, SEO-friendly, and gives us
realtime updates.

Our renderer takes lines of json as input and returns html strings as its
output. Text, the universal interface!

render.js:

``` js
var through = require('through');
var hyperglue = require('hyperglue');
var fs = require('fs');
var html = fs.readFileSync(__dirname + '/static/row.html', 'utf8');

module.exports = function () {
    return through(function (line) {
        try { var row = JSON.parse(line) }
        catch (err) { return this.emit('error', err) }
        
        this.queue(hyperglue(html, {
            '.who': row.who,
            '.message': row.message
        }).outerHTML);
    });
};
```

We can use [brfs](http://github.com/substack/brfs) to inline the
`fs.readFileSync()` call for browser code
and [hyperglue](https://github.com/substack/hyperglue) to update html based on
css selectors. You don't need to use hyperglue necessarily here; anything that
can return a string with html in it will work.

The `row.html` used is just a really simple stub thing:

row.html:

``` html
<div class="row">
  <div class="who"></div>
  <div class="message"></div>
</div>
```

The server will just use [slice-file](https://github.com/substack/slice-file) to
keep everything simple. [slice-file](https://github.com/substack/slice-file) is
little more than a glorified `tail/tail -f` api but the interfaces map well to
databases with regular results plus a changes feed like couchdb.

server.js:

``` js
var http = require('http');
var fs = require('fs');
var hyperstream = require('hyperstream');
var ecstatic = require('ecstatic')(__dirname + '/static');

var sliceFile = require('slice-file');
var sf = sliceFile(__dirname + '/data.txt');

var render = require('./render');

var server = http.createServer(function (req, res) {
    if (req.url === '/') {
        var hs = hyperstream({
            '#rows': sf.slice(-5).pipe(render())
        });
        hs.pipe(res);
        fs.createReadStream(__dirname + '/static/index.html').pipe(hs);
    }
    else ecstatic(req, res)
});
server.listen(8000);

var shoe = require('shoe');
var sock = shoe(function (stream) {
    sf.follow(-1,0).pipe(stream);
});
sock.install(server, '/sock');
```

The first part of the server handles the `/` route and streams the last 5 lines
from `data.txt` into the `#rows` div.

The second part of the server handles realtime updates to `#rows` using
[shoe](http://github.com/substack/shoe), a simple streaming websocket polyfill.

Next we can write some simple browser code to populate the realtime updates
from [shoe](http://github.com/substack/shoe) into the `#rows` div:

``` js
var through = require('through');
var render = require('./render');

var shoe = require('shoe');
var stream = shoe('/sock');

var rows = document.querySelector('#rows');
stream.pipe(render()).pipe(through(function (html) {
    rows.innerHTML += html;
}));
```

Just compile with [browserify](http://browserify.org) and
[brfs](http://github.com/substack/brfs):

```
$ browserify -t brfs browser.js > static/bundle.js
```

And that's it! Now we can populate `data.txt` with some silly data:

```
$ echo '{"who":"substack","message":"beep boop."}' >> data.txt
$ echo '{"who":"zoltar","message":"COWER PUNY HUMANS"}' >> data.txt
```

then spin up the server:

```
$ node server.js
```

then navigate to `localhost:8000` where we will see our content. If we add some
more content:

```
$ echo '{"who":"substack","message":"oh hello."}' >> data.txt
$ echo '{"who":"zoltar","message":"HEAR ME!"}' >> data.txt
```

then the page updates automatically with the realtime updates, hooray!

We're now using exactly the same rendering logic on both the client and the
server to serve up SEO-friendly, indexable realtime content. Hooray!
