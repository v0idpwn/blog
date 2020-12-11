---
title: "Writing a simple mailing service in Rust"
date: 2020-12-11T06:55:02-03:00
draft: false
---

Generally, when working with microservices you don't want to have every service
sending emails separately. Building a mailing service is pretty simple and
today we are gonna do it with Rust, because other languages are too mainstream.

We will first develop the application, then dockerize it and then finally use
it on a docker-compose file.

This application was written with **integration tests** in mind. In production
you should probably use something more robust.

### Step 1: HTTP
We must be able to receive HTTP requests. We will be using `actix` because it
is fast and has a better API than Rocket. We will also be using `serde` to
serialize data in JSON so we can talk with the external world.

So edit your `Cargo.toml` and add the following dependencies:
```toml
actix-web = "2"
actix-rt = "1"
serde = "1"
```

### Step 2: Receiving requests
The app is pretty straightforward, we would only need one endpoint: `send`,
which should receive POST requests with email data. Yet, I'll be adding 
a healthcheck endpoint where we may check if the app is ok. For now it will
just return "OK" to every request.

We can do it on our `src/main.rs` file like this:
```rust
use actix_web::{get, App, HttpServer, Responder};

#[get("/health")]
async fn health() -> impl Responder {
    format!("OK")
}

#[actix_rt::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| App::new().service(health))
        .bind("127.0.0.1:8080")?
        .run()
        .await
}
```
Requests can be declared with the `#[method(path)]` macro, and our `main`
function starts our server.

If we run it try sending a request, we may check it works (I'm using httpie to
test):
```
$ http GET localhost:8080/health
HTTP/1.1 200 OK
content-length: 2
content-type: text/plain; charset=utf-8
date: Sun, 12 Apr 2020 06:21:47 GMT

OK
```

### Step 3: A mail server for testing purposes
We won't be sending real emails in this tutorial. I'd rather run a SMTP server
locally:

Just run `docker pull mailhog/mailhog && docker run -p 8025:8025 -p
1025:1025 mailhog/mailhog`.

If everything goes well, you can find a nifty mailbox on `localhost:8025`. The
port 1025 is where we should send our emails. 

### Step 4: Sending emails
Now we start dwelling in our true domain: sending emails.  There is a quite 
popular Rust library to send emails, it is called `lettre`. The usage is fairly 
easy and thus it is seems like a good option.

At first, I had a struggle with the documentation: the library still didn't
reached the 1.0 version and had some breaking changes. We will be using version
0.9.2, which is the current supported version.

Add it on your `Cargo.toml`:
```toml
lettre = "0.9"
lettre_email= "0.9"
```

The `lettre_email` library is where the email builder lives, and `lettre`
handles the client and sending stuff.

Lets try sending a hardcoded email upon a request on new endpoint:

```rust
#[post("/api/v1/send")]
async fn send() -> impl Responder {
    let email = EmailBuilder::new()
        .to(("paleking@hallow.nest", "Pale King"))
        .from("theknight@example.com")
        .subject("Example email")
        .html("We got html support")
        .build()
        .unwrap();

    let mut mailer = SmtpClient::new(("localhost", 1025), ClientSecurity::None)
        .unwrap()
        .smtp_utf8(true)
        .transport();

    let result = mailer.send(email.into());
    assert!(result.is_ok());
    mailer.close();

    format!("OK")
}
```

This works, right? I received an email on my mailhog. But we must improve a lot
to be a mailing API. Next step? Receiving user input!

### Step 5: Internal representations and user input
We are gonna define structs for request bodies and responses. To send an email
we need the followind parameters: `to`, `from`, `subject`, `html`. So, lets
define a struct for this purpose:

```rust
#[derive(Deserialize)]
struct RequestBody {
    to: String,
    from: String,
    subject: String,
    html: String
}  
```

You should also import `Deserialize` from `serde`.

We may now receive them on the request body:

```rust
async fn send(body: web::Json<RequestBody>) -> impl Responder {
     let email = EmailBuilder::new()
         .to(body.to.clone())
         .from(body.from.clone())
         .subject(body.subject.clone())
         .html(body.html.clone())
         .build()       
         .unwrap();     
 
     let mut mailer = SmtpClient::new(("localhost", 1025), ClientSecurity::None)
         .unwrap()
         .smtp_utf8(true)
         .transport();
 
     let result = mailer.send(email.into());
     assert!(result.is_ok());
     mailer.close();
 
     format!("OK")
 }
```

This is gonna do fine when everything goes fine, but we are panic-ing! This
means that when something bad happens we are crashing instead of returning an
error message. *Hint: we are using clone because in rust we can't just pass
strings around, as they're references for the heap.*

### Step 6: Error handling
To handle the possible errors we may just match the build result instead of
unwrapping it. So the tail of our function becomes:
```rust
let response = 
    match email {
        Ok(e) => {
          let result = mailer.send(e.into());
          match result.is_ok() {
              true => format!("OK"),
              false => format!("error on sending routine")
          }
        },
        Err(err) => format!("{}", err)
    };

mailer.close();

response
```

Yet we are still not responding JSON, we can change this behavior pretty easily
creating some helper functions:
```rust
   fn ok_resp() -> HttpResponse {
       HttpResponse::Ok().json(Response{
           status: "ok".to_string(),
           message: "email sent".to_string()
       })
   }
   
   fn error_resp(message: &str) -> HttpResponse {
       HttpResponse::Ok().json(Response {
           status: "error".to_string(),
           message: message.to_string()
       })
   }
```

And changing our function to use them:
```rust
  let response = 
      match email {
          Ok(e) => {
            let result = mailer.send(e.into());
            match result.is_ok() {
                true => ok_resp(),
                false => error_resp("error on sending routine")
            }
          },
          Err(_) => error_resp("invalid params"),
      };

  mailer.close();

  response
```

This is pretty simple, but should work. Actix takes care of returning a `400
- Bad Request` when the request body doesn't match our type specification.

### Step 6: Logging
Monitoring applications is hard, and may become harder if you don't have proper logging. Adding logging to our existing application is fairly easy. We must just append a logger to Actix and set up our log level. 

We will use `env_logger` to set the log level, so add to your `Cargo.toml`. It
is also important to note that the logs are dependent on the `RUST_LOG`
environment variable:

```toml
 env_logger = "0.7"
```

Then we must set it up in our main function:
```rust
  env_logger::init();
```

This will make lettre logs work, but we still have to setup Actix Logging, so
we change our app declaration:

```rust
  HttpServer::new(|| App::new()
                    .service(health)
                    .service(send)
                    .wrap(Logger::default())
            )
            .bind("127.0.0.1:8080")?
            .run()
            .await

```

### Step 7: Environment
We can't keep hitting `localhost:1025` forever. Once the application is
dockerized, this won't even hit mailhog. The server URL and other params must
be set on environment.

We can add this feature changing just 3 lines of code in our sending function:

```rust
   let host = env::var("SMTP_HOST").expect("SMTP_HOST environment is mandatory");
   let port : u16 = env::var("SMTP_PORT").expect("SMTP_PORT environment is mandatory").parse::<u16>().expect("SMTP_PORT must be a number");
                                                                                
   let mut mailer = SmtpClient::new((host.as_str(), port), ClientSecurity::None)
```

and importing `env`:
```rust
use std::env;
```

Now, our application must be started with the `SMTP_PORT` and `SMTP_HOST`
environment variables, as:

```bash
$ RUST_LOG=info SMTP_HOST="localhost" SMTP_PORT=25 cargo run
```

### Step 8: Dockerfile
This is the Dockerfile I used to build and create the image:

```
# ------------------------------------------------------------------------------
# Cargo Build Stage
# ------------------------------------------------------------------------------

FROM ekidd/rust-musl-builder:latest as cargo-build

RUN rustup target add x86_64-unknown-linux-musl

WORKDIR /usr/src/crab_mail

ADD --chown=rust:rust . ./

RUN cargo build --release

# ------------------------------------------------------------------------------
# Final Stage
# ------------------------------------------------------------------------------

FROM alpine:latest

RUN addgroup -g 1000 crab_mail

RUN adduser -D -s /bin/sh -u 1000 -G crab_mail crab_mail

WORKDIR /home/crab_mail/bin/

COPY --from=cargo-build /usr/src/crab_mail/target/x86_64-unknown-linux-musl/release/crab_mail .

RUN chown crab_mail:crab_mail crab_mail

USER crab_mail

CMD ["./crab_mail"]
```

It should build a pretty lightweight final image.

### Step 9: Running on docker-compose on a bigger app
I uploaded the image to docker-hub and now I may use it in other projects :)

The use case that motivated this was setting an API to send emails to a mailhog
container. The example is here:

```yml
crab_mail:
  image: v0idpwn/crab_mail:v0.1.4
  environment:
    - SMTP_HOST=mailhog
    - SMTP_PORT=25
    - RUST_LOG=info
  ports:
    - 7026:8080
  depends_on:
    - mailhog

mailhog:
  image: mailhog/mailhog
  command: ["-smtp-bind-addr", "0.0.0.0:25"]
  user: root
  expose:
    - 25
    - 8025
  ports:
    - 8025:8025
  healthcheck:
    test: echo | telnet 127.0.0.1 25
```

It would need some improvements (like TLS) to send over the internet, but for
my purpose it works well enough :)
