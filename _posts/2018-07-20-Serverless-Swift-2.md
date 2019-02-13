---
layout: post
title: Swift on the server... without the server (Part 2)
date: 2018-07-20
tags: [Swift, Serverless]
redirect_from:
  - /blog/2018/07/20/Serverless-Swift-2.html.html
---

In [part 1](/blog/2018/07/13/Serverless-Swift.html.html), I outlined why Serverless architecture, and Lambda in particular, could be a really great solution for "server side" Swift, and how to get the bare minimum of a hello world example working. But it takes more than getting a process to run to use Swift as a backend. In part 2, I'd like to build on that work and work towards a more production ready environment.

### Run and done isn't enough

We've been using the same technique AWS recomends for all of their non-native languages, which is to start a new child process on every request. This is *ok* for one off jobs like processing an image, but if we want to use Swift for our entire function bodies, once we start connecting to things like databases and using cachable resources, that will start to be a bottleneck. While Lambda encourages you to think about a function as handling a single request/job, for performance reasons the process is (sometimes) left running between excecutions.

The result will be quit a bit more complicated than our first iteration. However, we will be able to reuse both the JS and Swift code between functions.

So let's change our bootstraping around to support that. First thing, we want to start our child process as soon as the JS process starts. That's easy enough to do. JS excecutes code at the root of documents on launch, so anything outside of the `exports.handler` will only run once per launch.

```js
// index.js
const spawn = require("child_process").spawn;

const command = "libraries/ld-linux-x86-64.so.2";
const main = spawn(command, ["--library-path", "libraries", "./main"]);

exports.handler = (event, context, callback) => {
  // tbd
};
```

You might notice that a key difference here from our last iteration is that we are not passing in the input to the child process. Because the child process is going to be reused, we can't just dump JSON into it, and likewise, we can't just read it's output until it exits. Instead we need some way to indicate when the entire body of the request and response have been sent.

Sockets (at least the kind we are using) don't have a concept of complete messages. TCP is just a stream of bytes without boundaries. So we have to come up with a method that indicates those boundaries. A typical approach is to send a header of some kind that indicates how long the body of the message is going to be. HTTP for instance sends the body size in it's headers, and thend marks the start of the body with a double line break. But we can use a much simpler protocol. We'll use a fixed length header of 32bits/4bytes as an unsigned integer that is the length of the body to follow. The reciever will know to read at least 4 bytes, then read the count from that 4 bytes before handling the message.

> ### Sidenote:
> Theoretically you could just examine the JSON body we are sending to find the end of the message. For instance, count the number of "{", "}", "[" and "]" characters and if they match, assume we are at the end. That gets more complicated when you realize those characters might appear in a JSON string. It also means that if there was invalid JSON, it would fail slowly, timing out waiting for the rest of the document that would never come.

StdIn and Out our convenient sockets that are setup for us by default, but they become crowded by other traffic. In particular, it’s very easy for a stray print statement to throw off a carefully crafted stream to stdout. And Lambda uses StdOut and Err for logging. Instead, if we’re going to get serious about our inter process communication, we need a dedicated pipe. We could use TCP like a web server would, listening on a port, but because both of these processes are running on the same Unix host, and we don’t really want outside communication, we can use a Unix socket which is more efficient. It’s actually how the standard pipes are implemented.

We’ll setup the js bootstrap as the server so it can start listening and be ready to accept connections as soon as our swift process tries to connect. That looks like this:

```js
const server = net.createServer(socket => {
  // socket is a connection to our Swift child process
});
server.listen("/tmp/swift.sock");
```

To connect from Swift, we’ll use [BlueSocket](https://github.com/IBM-Swift/BlueSocket), which is a pleasant Swift wrapper around the low level C socket api. Unfortunately [Network.framework](https://developer.apple.com/documentation/network) isn't available on Linux 😢. SwiftNIO would work here possibility, but it’s significantly more complex, and it’s big feature, concurrency, won’t be much good since we’ll only be handling 1 request at a time.

```swift
let socket = try Socket.create(family: .unix, type: .stream, proto: .unix)
try socket.connect(to: "/tmp/swift.sock")

var buffer = Data()
func read(count: Int) throws -> Data {
	while buffer.count < count {
		var newData = Data()
		_ = try socket.read(into: &newData)
		buffer.append(newData)
	}
	
	let chunk = buffer.prefix(count)
	buffer.removeFirst(count)
	
	return chunk
}

// start trying to read the 4 byte header
let header = try self.read(count: 4)
```

When our JS handler gets a request (which should happen right after it's launched), it will encode it to JSON and send it over.

```js
exports.handler = (event, context, callback) => {
  var jsonBuffer = new Buffer(JSON.stringify(event), "binary");
  let countBuffer = new Buffer(4);
  countBuffer.writeUInt32BE(jsonBuffer.length, 0);
  socket.write(countBuffer);
  socket.write(jsonBuffer);
};
```

Even if you are familiar with JS in the browser, you may not have seen `Buffer` before. It's basically node's version of `Data`. Notice that we are writting the count as [big endian](https://en.wikipedia.org/wiki/Endianness) (`writeUInt32BE`), which is typically what you use for network communications. We'll need to make sure to read it that way in Swift.

```swift
let headerCount = header.withUnsafeBytes({ UInt32(bigEndian: $0.pointee) })
let data = try self.read(count: Int(count))
// parse data with either JSONDecoder or JSONSerialization
```

Cool, now we have our request event. Now we do... something. That part really depends on the function you are writting. But once the request if done, we need to do the wave and send things right back. We'll mostly do the same thing for the response, but there's one more thing we have to handle. A response might generate an error, which we'll need to communcate to our JS process. We'll extend our header with an extra byte at the beginning. 1 will indicate success while 2 will indicate an error.

```swift
func write(_ data: Data) throws {
	try self.socket.write(from: data)
}

func write<T: FixedWidthInteger>(_ value: T) throws {
	var value = value.bigEndian
	let bufSize = value.bitWidth / UInt8.bitWidth
	_ = try withUnsafeBytes(of: &value) { (pointer) in
		try self.socket.write(from: pointer.baseAddress!, bufSize: bufSize)
	}
}
try self.write(1 as UInt8)
try self.write(UInt32(jsonData.count))
try self.write(jsonData)
```

Again, `jsonData` is going to be dependent on the task you are doing and could be generated from either a `JSONEncoder` or `JSONSerialization`. The final Swift implimentation will need to be in an infinite loop so that it can handle multiple request. That would look something like this:

```swift
while true {
	let header = try self.read(count: 4)
	let headerCount = header.withUnsafeBytes({ UInt32(bigEndian: $0.pointee) })
	let data = try self.read(count: Int(count))
	
	// do something that creates jsonData
	
	try self.write(1 as UInt8)
	try self.write(UInt32(jsonData.count))
	try self.write(jsonData)
}
```

That way, as soon as we are done writting out our response we immediately turn around and start waiting for a new request.

Handling the response in JS is a little more complicated because it doesn't have synchronous socket reads. To be clear, under normal circumstances node's approach is far better and it's the way Network.framework and SwiftNIO work.

We'll need to instead listen for data as they come in. Again, just because we get a chunk of data doesn't mean that's the entire message, so we'll need to keep track of some state:

```js
const responseEmitter = new EventEmitter();
let messageType = null;
let messageLength = null;
let buffer = null;
socket.on("data", data => {
  buffer = buffer ? Buffer.concat([buffer, data]) : data;

  if (messageType === null && buffer.length >= 1) {
    messageType = buffer.readUInt8(0);
  }
  if (messageLength === null && buffer.length >= 5) {
    messageLength = buffer.readInt32BE(1);
  }
  if (messageType !== null && buffer.length >= 5 + messageLength) {
    const json = buffer.toString("utf8", 5, 5 + messageLength);
    const message = JSON.parse(json);

    if (messageType === 1) {
      responseEmitter.emit("success", message);
    } else {
      responseEmitter.emit("error", message);
    }

    // reset for the next message
    // because we use a serial request/response prototocol,
    // we don't have to worry about buffer having multiple messages
    messageType = null;
    messageLength = null;
    buffer = null;
  }
});
```

`responseEmitter` is used to notify the handler that a response has been recieved. `EventEmitter` works kind of like `NotificationCenter` in Swift.

At this point we have the basic building blocks to handle a persistent connection between the JS bootstrap and the Swift child process. Like I said, this is a lot more complicated but can be reused. I've wrapped all of this (along with better error handling) into package: [aws-lambda-swift-hook](https://github.com/davbeck/aws-lambda-swift-hook). Using that package, we can build a function with the following:

```swift
import Foundation
import Lambda

struct Input: Codable {
	var key1: String = ""
	var key2: String = ""
	var key3: String = ""
}

struct Output: Codable {
	var hello: String
}

start() { (context, input: Input, completion: (Result<Output>) -> Void) in
	completion(.success(Output(hello: "lambda")))
}
```

It could be cleaner if Swift supported [async/await](https://gist.github.com/lattner/429b9070918248274f25b714dcfc7619), but otherwise it's pretty straigtforward.

## Next time

In part 3 I hope to bring all of this together into a complete serverless project.