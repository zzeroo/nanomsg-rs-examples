# Nanomsg-rs Examples

> Nanomsg is a modern messaging library that is the successor to ZeroMQ, written in C by Martin Sustrik and colleagues. The nanomsg library is licensed under MIT/X11 license. "nanomsg" is a trademark of 250bpm s.r.o.
>
> -- <cite>From the README.md of the [Nanomsg.rs](http://thehydroimpulse.github.io/nanomsg.rs/nanomsg) project</cite>

- [Original Nanomsg project](http://nanomsg.org/)
- [Nanomsg.rs project](http://thehydroimpulse.github.io/nanomsg.rs/nanomsg)

This project tries to recreate the wonderful nanoms examples, writen in C from http://tim.dysinger.net/posts/2013-09-16-getting-started-with-nanomsg.html in plain Rust.

- [Getting Started with nanomsg by Tim Dysinger](http://tim.dysinger.net/posts/2013-09-16-getting-started-with-nanomsg.html)
- [dysinger's github.com repository](https://github.com/dysinger/nanomsg-examples)

# Getting Started

For a project of mine I have to work with nanomsg. Because I'm going to write the entire project in Rust I thought that it makes sense to port these instructions.

## Pipeline (A one-way pipe)
![](share/pipeline.png)

```rust
extern crate nanomsg;
use nanomsg::{Socket, Protocol};
use std::io::{Read, Write};

fn node0(url: String) {
    let mut socket = Socket::new(Protocol::Pull).unwrap();
    let mut request = String::new();
    let _ = socket.bind(&url); // let _ = means we don't need any return value stored somewere

    loop {
        match socket.read_to_string(&mut request) {
            Ok(_) => println!("NODE0: RECEIVED '{}'", request),
            Err(err) => {
                println!("NODE0: failed '{}'", err);
                break
            }
        }
        request.clear();
    }
}

fn node1(url: String, msg: String) {
    let mut socket = Socket::new(Protocol::Push).unwrap();
    socket.connect(&url).unwrap();

    match socket.write_all(&msg.as_bytes()) {
        Ok(_) => println!("NODE1: SENDING '{}'", &msg),
        Err(err) => {
            println!("NODE1: failed '{}'", err);
        }
    }
}

fn usage() {
    println!("Usage: pipeline node0|node1 <URL> <ARG> ...");
}

fn main() {
    let args: Vec<_> = std::env::args().collect();

    if args.len() < 2 {
        return usage()
    }

    match args[1].as_ref() {
        "node0" => {
            if args.len() > 1 {
                node0(args[2].clone())
            }
        }
        "node1" => {
            if args.len() > 2 {
                node1(args[2].clone(), args[3].clone())
            }
        }
        _ => usage(),
    }
}
```
### Build this example
```
cargo build --example pipeline
```
### Run this example
```
cargo run --example pipeline -- node0 ipc:///tmp/pipeline.ipc &
cargo run --example pipeline -- node1 ipc:///tmp/pipeline.ipc "Hello, World\!"
cargo run --example pipeline -- node1 ipc:///tmp/pipeline.ipc "Goodbye."
killall pipeline
```
### Expected output
```
NODE1: SENDING 'Hello, World!'
NODE0: RECEIVED 'Hello, World!'
NODE1: SENDING 'Goodbye.'
NODE0: RECEIVED 'Goodbye.'
```

## Request/Reply (I ask, you answer)
![](share/request_reply.png)

```rust
extern crate nanomsg;
extern crate time;
use nanomsg::{Socket, Protocol};
use std::io::{Read, Write};
use std::thread;
use std::time::{Duration};


fn date() -> String {
    let time = time::get_time();
    let local = time::at(time);
    local.asctime().to_string()
}

fn node0(url: String) {
    let mut socket = Socket::new(Protocol::Rep).unwrap();
    let mut endpoint = socket.bind(&url).unwrap();

    let mut request = String::new();

    loop {
        match socket.read_to_string(&mut request) {
            Ok(_) => {
                println!("NODE0: RECEIVED DATE REQUEST");
                let reply = format!("{}", date());
                match socket.write_all(reply.as_bytes()) {
                    Ok(..) => { println!("NODE0: SENDING DATE {}", reply); },
                    Err(err) => {
                        println!("NODE0: Failed to send reply '{}'", err);
                        break
                    }
                }
                request.clear();
            },
            Err(err) => {
                panic!("NODE0: Problem while reading: '{}'", err);
            }
        }
        thread::sleep(Duration::from_millis(400));
    }
    let _ = endpoint.shutdown();
    drop(socket);
}

fn node1(url: String) {
    let mut socket = Socket::new(Protocol::Req).unwrap();
    let mut endpoint = socket.connect(&url).unwrap();

    let mut request = String::new();

    println!("NODE1: SENDING DATE REQUEST {}", "DATE");
    match socket.write_all("DATE".as_bytes()) {
        Ok(_) => {
            match socket.read_to_string(&mut request) {
                Ok(_) => { println!("NODE1: RECEIVED DATE {}", request); },
                Err(err) => { println!("NODE1: Failed to read replay '{}'", err); }
            }
        },
        Err(err) => { println!("NODE1: Failed to write DATE request '{}'", err); }
    }
    let _ = endpoint.shutdown();
}


fn usage() {
    println!("Usage: request_reply node0|node1 <URL>");
}

fn main() {
    let args: Vec<_> = std::env::args().collect();

    if args.len() < 2 {
        return usage()
    }

    match args[1].as_ref() {
        "node0" => node0(args[2].clone()),
        "node1" => node1(args[2].clone()),
        _ => usage(),
    }
}
```

### Build this example
```
cargo build --example request_reply
```
### Run this example
```
cargo run --example request_reply -- node0 ipc:///tmp/request_reply.ipc &
cargo run --example request_reply -- node1 ipc:///tmp/request_reply.ipc
killall request_reply
```
### Expected output, your date is certainly another
```
NODE1: SENDING DATE REQUEST DATE
NODE0: RECEIVED DATE REQUEST
NODE0: SENDING DATE Mon May 30 12:02:01 2016
NODE1: RECEIVED DATE Mon May 30 12:02:01 2016
```

# Pair (Two Way Radio)
![](share/request_reply.png)

```rust
extern crate nanomsg;
extern crate time;
use nanomsg::{Socket, Protocol};
use std::io::{Read, Write};
use std::thread;
use std::time::{Duration};


fn send_name(socket: &mut Socket, name: &str) {
    match socket.write_all(name.as_bytes()) {
        Ok(_) => {println!("{}: SENDING \"{}\"", name, name);},
        Err(err) => println!("Could not send: {}", err)
    }
}

fn receive_name(socket: &mut Socket, name: &str) -> String {
    let mut result = String::new();
    match socket.read_to_string(&mut result) {
        Ok(_) => println!("{}: RECEIVED \"{}\"", name, &result),
        Err(_) => {},
    }

    result
}

fn send_receive(socket: &mut Socket, name: &str) {
    let _ = socket.set_receive_timeout(100); // without this receive_name would block and never time out
    loop {
        receive_name(socket, name);
        thread::sleep(Duration::from_millis(1000));
        send_name(socket, name);
    }
}

fn node0(url: String) {
    let mut socket = Socket::new(Protocol::Pair).unwrap();
    let mut endpoint = socket.bind(&url).unwrap();

    send_receive(&mut socket, "node0");

    let _ = endpoint.shutdown();
}

fn node1(url: String) {
    let mut socket = Socket::new(Protocol::Pair).unwrap();
    let mut endpoint = socket.connect(&url).unwrap();

    send_receive(&mut socket, "node1");

    let _ = endpoint.shutdown();
}

fn usage() {
    println!("Usage: pair node0|node1 <URL>");
}

fn main() {
    let args: Vec<_> = std::env::args().collect();

    if args.len() < 2 {
        return usage()
    }

    match args[1].as_ref() {
        "node0" => node0(args[2].clone()),
        "node1" => node1(args[2].clone()),
        _ => usage(),
    }
}
```

### Build this example
```
cargo build --example pair
```
### Run this example
```
cargo run --example pair -- node0 ipc:///tmp/pair.ipc &
cargo run --example pair -- node1 ipc:///tmp/pair.ipc &
sleep 3
killall pair
```
### Expected output
```
node0: SENDING "node0"
node1: SENDING "node1"
node1: RECEIVED "node0"
node0: RECEIVED "node1"
node1: SENDING "node1"
node1: RECEIVED "node0"
node0: SENDING "node0"
node0: RECEIVED "node1"
```

# Pub/Sub (Topics & Broadcast)
![](share/pub_sub.png)

```rust
extern crate nanomsg;
extern crate time;
use nanomsg::{Socket, Protocol};
use std::io::{Read, Write};
use std::thread;
use std::time::{Duration};


fn date() -> String {
    let time = time::get_time();
    let local = time::at(time);
    local.asctime().to_string()
}

fn server(url: String) {
    let mut socket = Socket::new(Protocol::Pub).unwrap();
    let _ = socket.bind(&url).unwrap();

    loop {
        let date = date();

        match socket.write_all(&date.as_bytes()) {
            Ok(_) => {
                println!("SERVER: PUBLISHING DATE {}", date);
                thread::sleep(Duration::from_millis(100));
            },
            Err(err) => panic!("{}", err),
        }
    }
}

fn client(url: String, name: String) {
    let mut socket = Socket::new(Protocol::Sub).unwrap();
    let topic = "";
    match socket.subscribe(topic) {
            Ok(_) => {},
            Err(err) => panic!("{}", err)
    }
    let _ = socket.connect(&url).unwrap();

    let mut buffer = String::new();

    loop {
        match socket.read_to_string(&mut buffer) {
            Ok(_) => {
                println!("CLIENT ({}): RECEIVED {}", name, buffer);
                buffer.clear();
            },
            Err(err) => { panic!("{}", err); }
        }
    }
}

fn usage() {
    println!("Usage: pubsub server|client <URL> <ARG> ...");
}

fn main() {
    let args: Vec<_> = std::env::args().collect();

    if args.len() < 2 {
        return usage()
    }

    match args[1].as_ref() {
        "server" => server(args[2].clone()),
        "client" => client(args[2].clone(), args[3].clone()),
        _ => usage(),
    }
}
```

### Build this example
```
cargo build --example pubsub
```
### Run this example
```
cargo run --example pubsub -- server ipc:///tmp/pubsub.ipc & sleep 2 &
cargo run --example pubsub -- client ipc:///tmp/pubsub.ipc  client0 &
cargo run --example pubsub -- client ipc:///tmp/pubsub.ipc  client1 &
cargo run --example pubsub -- client ipc:///tmp/pubsub.ipc  client2 &
sleep 5
killall pubsub
```
### Expected output (maybe in different order)
```
SERVER: PUBLISHING DATE Tue May 31 13:00:43 2016
CLIENT (client0): RECEIVED Tue May 31 13:00:43 2016
CLIENT (client1): RECEIVED Tue May 31 13:00:43 2016
CLIENT (client2): RECEIVED Tue May 31 13:00:43 2016
SERVER: PUBLISHING DATE Tue May 31 13:00:44 2016
CLIENT (client1): RECEIVED Tue May 31 13:00:44 2016
CLIENT (client2): RECEIVED Tue May 31 13:00:44 2016
CLIENT (client0): RECEIVED Tue May 31 13:00:44 2016
```


# Survey (Everybody Votes)
![](share/survey.png)

```rust
extern crate nanomsg;
extern crate time;
use nanomsg::{Socket, Protocol};
use std::io::{Read, Write};
use std::thread;
use std::time::{Duration};


fn date() -> String {
    let time = time::get_time();
    let local = time::at(time);
    local.asctime().to_string()
}

fn server(url: String) {
    let mut socket = Socket::new(Protocol::Surveyor).unwrap();
    let _ = socket.bind(&url).unwrap();
    // Wait for connections
    thread::sleep(Duration::from_millis(1000));
    match socket.write_all("DATE".as_bytes()) {
        Ok(_) => {
            println!("SERVER: SENDING DATE SURVEY REQUEST");
        },
        Err(err) => { panic!("{}", err); },
    }


    loop {
        let mut buffer = String::new();

        match socket.read_to_string(&mut buffer) {
            Ok(_) => {
                println!("SERVER: RECEIVED \"{}\" SURVEY RESPONSE", buffer);
                buffer.clear();
            },
            Err(_) => { break; }
        }
    }
}

fn client(url: String, name: String) {
    let mut socket = Socket::new(Protocol::Respondent).unwrap();
    let _ = socket.connect(&url).unwrap();

    loop {
        let mut buffer = String::new();

        match socket.read_to_string(&mut buffer) {
            Ok(_) => {
                println!("CLIENT ({}): RECEIVED \"{}\" SURVEY REQUEST", name, buffer);
                buffer.clear();
                let date = date();
                match socket.write_all(&date.as_bytes()) {
                    Ok(_) => { println!("CLIENT ({}): SENDING DATE SURVEY RESPONSE", name); },
                    Err(err) => {
                        println!("CLIENT ({}) could not send date survey response: {}", name, err);
                    }
                }
            },
            Err(_) => { break; }
        }
    }
}

fn usage() {
    println!("Usage: survey server|client <URL> <ARG> ...");
}

fn main() {
    let args: Vec<_> = std::env::args().collect();

    if args.len() < 2 {
        return usage()
    }

    match args[1].as_ref() {
        "server" => server(args[2].clone()),
        "client" => client(args[2].clone(), args[3].clone()),
        _ => usage(),
    }
}
```

### Build this example
```
cargo build --example survey
```
### Run this example
```
cargo run --example survey -- server ipc:///tmp/survey.ipc &
cargo run --example survey -- client ipc:///tmp/survey.ipc  client0 &
cargo run --example survey -- client ipc:///tmp/survey.ipc  client1 &
cargo run --example survey -- client ipc:///tmp/survey.ipc  client2 &
sleep 3
killall -q cargo survey &>/dev/null
```
### Expected output
```
SERVER: SENDING DATE SURVEY REQUEST
CLIENT (client0): RECEIVED "DATE" SURVEY REQUEST
CLIENT (client1): RECEIVED "DATE" SURVEY REQUEST
CLIENT (client2): RECEIVED "DATE" SURVEY REQUEST
CLIENT (client1): SENDING DATE SURVEY RESPONSE
CLIENT (client0): SENDING DATE SURVEY RESPONSE
CLIENT (client2): SENDING DATE SURVEY RESPONSE
SERVER: RECEIVED "Tue May 31 17:32:35 2016" SURVEY RESPONSE
SERVER: RECEIVED "Tue May 31 17:32:35 2016" SURVEY RESPONSE
SERVER: RECEIVED "Tue May 31 17:32:35 2016" SURVEY RESPONSE
```

#Bus (Routing)
![](share/bus.png)

```rust
extern crate nanomsg;
extern crate time;
use nanomsg::{Socket, Protocol};
use std::io::{Read, Write};
use std::thread;
use std::time::{Duration};


fn node(args: Vec<String>) {
    let mut socket = Socket::new(Protocol::Bus).unwrap();
    let _ = socket.bind(&args[2]).unwrap();

    let mut iter = args.iter().skip(3);

    loop {
        match iter.next() {
            Some(url) => {
                match socket.connect(&url) {
                    Ok(_) => {}
                    Err(err) => { panic!("{}", err); }
                }
            }
            None => break,
        }
    }
    thread::sleep(Duration::from_millis(1000));
    let _ = socket.set_receive_timeout(100);

    match socket.write_all(&args[1].as_bytes()) {
        Ok(_) => {
            println!("{}: SENDING '{}' ONTO BUS", &args[1], &args[1]);
        }
        Err(err) => { panic!("{}", err); }
    }

    loop {
        let mut buffer = String::new();

        match socket.read_to_string(&mut buffer) {
            Ok(_) => {
                println!("{}: RECEIVED '{}' FROM BUS", args[1], buffer);
                buffer.clear();
            }
            Err(_) => { }
        }
    }
}


fn usage() {
    println!("Usage: bus <NODE_NAME> <URL> <URL> ...");
}

fn main() {
    let args: Vec<_> = std::env::args().collect();

    if args.len() >= 3 {
        node(args);
    } else {
        return usage()
    }
}
```

### Build this example
```
cargo build --example bus
```
### Run this example
```
cargo run --example bus -- node0 ipc:///tmp/node0.ipc ipc:///tmp/node1.ipc ipc:///tmp/node2.ipc &
cargo run --example bus -- node1 ipc:///tmp/node1.ipc ipc:///tmp/node2.ipc ipc:///tmp/node3.ipc &
cargo run --example bus -- node2 ipc:///tmp/node2.ipc ipc:///tmp/node3.ipc &
cargo run --example bus -- node3 ipc:///tmp/node3.ipc ipc:///tmp/node0.ipc &
sleep 5
killall -q cargo bus &>/dev/null
```
### Expected output (maybe in different order)
```
node3: SENDING 'node3' ONTO BUS
node2: SENDING 'node2' ONTO BUS
node2: RECEIVED 'node3' FROM BUS
node3: RECEIVED 'node2' FROM BUS
node1: SENDING 'node1' ONTO BUS
node2: RECEIVED 'node1' FROM BUS
node3: RECEIVED 'node1' FROM BUS
node1: RECEIVED 'node3' FROM BUS
node1: RECEIVED 'node2' FROM BUS
node0: SENDING 'node0' ONTO BUS
node1: RECEIVED 'node0' FROM BUS
node0: RECEIVED 'node3' FROM BUS
node0: RECEIVED 'node2' FROM BUS
node2: RECEIVED 'node0' FROM BUS
node3: RECEIVED 'node0' FROM BUS
node0: RECEIVED 'node1' FROM BUS
```


<!--
create a local html file from markdown `pandoc --highlight-style=tango -s -f markdown -t html5 -o README.html README.md`
-->
