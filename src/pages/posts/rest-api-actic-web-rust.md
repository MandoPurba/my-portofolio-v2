---
layout: ../../layouts/post.astro
title: "REST APIs with Actix Web in Rust"
pubDate: 2023-12-23
description: "Actix Web is a compact, efficient, and robust asynchronous web framework in Rust, designed for creating APIs and web applications. It leverages Tokio and the futures crate internally. Actix Web also offers a synchronous API that integrates smoothly with its asynchronous counterpart. Additionally, it includes built-in support for logging, static file serving, TLS, HTTP/2, and various other features."
author: "romando"
isPinned: true
excerpt: Actix Web is a compact, efficient, and robust asynchronous web framework in Rust, designed for creating APIs and web applications. It leverages Tokio and the futures crate internally. Actix Web also offers a synchronous API that integrates smoothly with its asynchronous counterpart. Additionally, it includes built-in support for logging, static file serving, TLS, HTTP/2, and various other features.
image:
  src:
  alt:
tags: ["rust", "actix-web", "api", "web-development"]
---

# REST APIs with Actix Web in Rust
![img.png](img.png)

## INTRODUCTION
Actix Web is a compact, efficient, and robust asynchronous web framework in Rust, designed for creating APIs and web applications. It leverages Tokio and the futures crate internally. Actix Web also offers a synchronous API that integrates smoothly with its asynchronous counterpart. Additionally, it includes built-in support for logging, static file serving, TLS, HTTP/2, and various other features.
REST (Representational State Transfer) is a set of architectural principles for building APIs. A RESTful API adheres to these principles, ensuring compliance with the REST architecture.
- Client requests made over a RESTful API include the following components:
- Endpoint: A URI that expose the application's resource on the web.
- HTTP Method: Specifies the operation to perform POST, GET, PUT, DELETE.
- Header: Contains authentication credentials.
- Body (optional) : Include data or additional information.

This guide explains how to build a RESTful API in Rust using Actix Web. This sample application allows users to perform Create, Read, Update, and Delete (CRUD) operations on book objects stored in a database.

## SETUP
Initialize the project crate using Cargo:
```bash
cargo new actix_rust --bin
```
Switch to the newly created directory:
```bash
cd actix_rust
```
Open the Cargo.toml file, and add the following dependencies:
```toml
[package]
name = "actix_rust"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4.8.0"
serde = { version = "1.0.204", features = ["derive"] }
serde_json = "1.0.120"
```

## Import Libraries
Open the **_main.rs_** file in the **_src/_** directory, and overwrite its contents the following code
```rust
use actix_web::body::BoxBody;
use actix_web::http::header::ContentType;
use actix_web::http::StatusCode;
use actix_web::{
    delete, get, post, put, web, App, HttpRequest, HttpResponse, HttpServer, Responder,
    ResponseError,
};
use serde::{Deserialize, Serialize};
use std::fmt::{Display, Formatter};
use std::sync::{Arc, Mutex};
```

## Define Data Structures
**Create the book data structure:**
```rust
#[derive(Serialize, Deserialize)]
struct Book {
    id: String,
    title: String,
    description: String
}
```
The derive macro used on the Book struct enables Serde to automatically generate serialization and deserialization implementations for the Book struct.

**Implement Responder for Book**

To return the Book type directly as an **_HttpResponse_**, implement the Responder trait.
```rust
// Implement Responder Trait for Book
impl Responder for Book {
    type Body = BoxBody;

    fn respond_to(self, _req: &HttpRequest) -> HttpResponse<Self::Body> {
        let res_body = serde_json::to_string(&self).unwrap();
        // Create HttpResponse and set Content Type
        HttpResponse::Ok()
            .content_type(ContentType::json())
            .body(res_body)
    }
}
```
The Responder trait requires a function called respond_to, which converts self into an **_HttpResponse_**.
```rust
type Body = BoxBody;
```
This assigns a BoxBody type to the associated type, Body. The **_respond_to_** function takes two parameters: self and **_HttpRequest_**, and returns an **_HttpResponse_** of type **_Self::Body_**.

The Book struct implements the Serialize trait and is converted into JSON using the **_to_string_** function from **_serde_json_**.
```rust
let res_body = serde_json::to_string(&self).unwrap();
```
This serializes the Book struct into JSON format and assigns the serialized data to a variable called **_res_body_**. Then, it constructs an **_OK HttpResponse_** (status code 200) with the **_content-type_** set to **JSON** using the **_content_type_** method of **_HttpResponse_**, and sets the body of the response to the serialized data.
```rust
HttpResponse::Ok()
            .content_type(ContentType::json())
            .body(res_body)
```
The Book type can now be returned directly from a handler function, as it implements the Responder trait.

**Define Custom Error Struct**

The application needs to be able to send a custom error message if a user requests or tries to delete a book id that does not exist. Implement an ErrNoId struct, that holds an id and an error message.
```rust
#[derive(Debug, Serialize)]
struct ErrNoId {
    id: String,
    err: String
}
```
The derive macro enables the struct’s serialization into JSON using serde_json

**Implement ResponseError Trait**

Custom error responses in Actix Web must implement the ResponseError trait, which requires two methods — **status_code** and **error_response**
```rust
// Implement ResponseError for ErrNoId
impl ResponseError for ErrNoId {
    fn status_code(&self) -> StatusCode {
        StatusCode::NOT_FOUND
    }

    fn error_response(&self) -> HttpResponse<BoxBody> {
        let body = serde_json::to_string(&self).unwrap();
        let res = HttpResponse::new(self.status_code());
        res.set_body(BoxBody::new(body))
    }
}
```
The status_code function returns `StatusCode::NOT_FOUND` (status code 404)

**Implement Display Trait For ErrNoId Struct**

ResponseError requires a trait bound of `fmt::Debug + fmt::Display`. Using the `#[derive(Debug, Serialize)]` macro on ErrNoId, the Debug trait bound satisfies, but the Display doesn't. Implement the Display trait:
```rust
// Implement Display for ErrNoId
impl Display for ErrNoId {
    fn fmt(&self, f: &mut Formatter<'_>) -> std::fmt::Result {
        write!(f, "{:?}", self)
    }
}
```

**Define AppState Struct**

All the routes and resources within the same scope share an application state. Define a struct that holds the shared data:
```rust
// Create AppState
struct AppState {
    books: Mutex<Vec<Book>>
}
```


## Create Route Handlers
Handler functions in Actix Web are async to enable asynchronous processing.

Create route handler functions for each of the endpoints defined in the table earlier

**[POST] /books**
```rust
// Handler Create Book
#[post("/books")]
async fn post_book(req: web::Json<Book>, data: web::Data<AppState>) -> impl Responder {
    let new_book = Book {
        id: String::from(&req.id),
        title: String::from(&req.title),
        description: String::from(&req.description)
    };
    let response = serde_json::to_string(&new_book).unwrap();
    let mut books = data.books.lock().unwrap();
    books.push(new_book);
    HttpResponse::Created()
        .content_type(ContentType::json())
        .body(response)
}
```

**[GET] /books**
```rust
// handler get all books
#[get("/books")]
async fn get_books(data: web::Data<AppState>) -> impl Responder {
    let books = data.books.lock().unwrap();
    let response = serde_json::to_string(&(*books)).unwrap();
    HttpResponse::Ok()
        .content_type(ContentType::json())
        .body(response)
}
```

**[GET] /books/{id}**
```rust
// Handler get a book with the corresponding id
#[get("/books/{id}")]
async fn get_book(id: web::Path<String>, data: web::Data<AppState>) -> impl Responder {
    let book_id: String = id.into_inner();
    let books = data.books.lock().unwrap();
    let book_data = books.iter().find(|x| x.id == book_id);

    match book_data {
        None => {
            let response = ErrNoId {
                id: book_id,
                err: String::from("Book Not Found")
            };
            Err(response)
        },
        Some(book) => {
            let response = serde_json::to_string(book).unwrap();
            Ok(
                HttpResponse::Ok()
                    .content_type(ContentType::json())
                    .body(response)
            )
        }
    }
}
```

**[PUT] /books/{id}**
```rust
// Handler update the books with the corresponding id
#[put("/books/{id}")]
async fn update_book(id: web::Path<String>, req: web::Json<Book>, data: web::Data<AppState>) -> impl Responder {
    let book_id: String = id.into_inner();
    let new_book = Book {
        id: String::from(&req.id),
        title: String::from(&req.title),
        description: String::from(&req.description),
    };
    let mut books = data.books.lock().unwrap();
    let id_index = books.iter().position(|x| x.id == book_id);
    match id_index {
        None => {
            let response = ErrNoId {
                id: book_id,
                err: String::from("Book not found")
            };
            Err(response)
        }
        Some(id) => {
            let response = serde_json::to_string(&new_book).unwrap();
            books[id] = new_book;
            Ok(HttpResponse::Ok()
                .content_type(ContentType::json())
                .body(response)
            )
        }
    }
}
```

**[DELETE] /books/{id}**
```rust
// Handler delete book with the corresponding id
#[delete("/books/{id}")]
async fn delete_book(id: web::Path<String>, data: web::Data<AppState>) -> impl Responder {
    let book_id = id.into_inner();
    let mut books = data.books.lock().unwrap();
    let id_index = books.iter().position(|x| x.id == book_id);
    match id_index {
        None => {
            let response = ErrNoId {
                id: book_id,
                err: String::from("Book Not Found")
            };
            Err(response)
        }
        Some(id) => {
            let delete_book = books.remove(id);
            Ok(delete_book)
        }
    }
}
```

## Create Server
Create the server and configure the routes:
```rust
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let app_state = web::Data::new(AppState{
        books: Arc::new(Mutex::new(vec![
            Book {
                id: String::from("A0001"),
                title: String::from("Atomic Habits"),
                description: String::from("Small changes give extraordinary results.")
            },
            Book {
                id: String::from("A0002"),
                title: String::from("The Psychology of Money"),
                description: String::from("Managing money well has less to do with your intelligence and more to do with your behavior.")
            }
        ]))
    });
    HttpServer::new(move || {
        App::new()
            .app_data(app_state.clone())
            .service(post_book)
            .service(get_books)
            .service(get_book)
            .service(update_book)
            .service(delete_book)
    })
    .bind(("127.0.0.1", 8000))?
    .run()
    .await
}
```

The macro `#[actix_web::main]` marks an async main function as the Actix system's entry point. This macro executes the async main function with the Actix runtime. The main function returns a type `std::io::Result<()>`.

To create the application’s shared mutable state, `use web::Data::new()`:
```rust
et app_state = web::Data::new(AppState{
        books: Arc::new(Mutex::new(vec![
            Book {
                id: String::from("A0001"),
                title: String::from("Atomic Habits"),
                description: String::from("Small changes give extraordinary results.")
            },
            Book {
                id: String::from("A0002"),
                title: String::from("The Psychology of Money"),
                description: String::from("Managing money well has less to do with your intelligence and more to do with your behavior.")
            }
        ]))
    });
```
The vector of tickets is also populated with a few entries.

Actix Web servers build around the App instance, which registers routes for resources and middleware. It also stores the application state shared across all handlers.

`HttpServer::new()` takes an application factory as an argument rather than an instance.

```rust
App::new()
    .app_data(app_state.clone())
    .service(post_book)
    .service(get_books)
    .service(get_book)
    .service(update_book)
    .service(delete_book)
```

This creates an application builder using new() and sets the application level shared mutable data using the app_data method. Register the handlers to the App using the service method.

The bind method of HttpServer binds a socket address to the server. To run the server, call the `run()` method. The server must then be await'ed or spawn'ed to start processing requests and runs until it receives a shutdown signal.

## Final Code
For reference, the final code in the src/main.rs file:

```rust
use actix_web::body::BoxBody;
use actix_web::http::header::ContentType;
use actix_web::http::StatusCode;
use actix_web::{
    delete, get, post, put, web, App, HttpRequest, HttpResponse, HttpServer, Responder,
    ResponseError,
};
use serde::{Deserialize, Serialize};
use std::fmt::{Display, Formatter};
use std::sync::{Arc, Mutex};

#[derive(Serialize, Deserialize)]
struct Book {
    id: String,
    title: String,
    description: String,
}

// Implement Responder Trait for Book
impl Responder for Book {
    type Body = BoxBody;

    fn respond_to(self, _req: &HttpRequest) -> HttpResponse<Self::Body> {
        let res_body = serde_json::to_string(&self).unwrap();
        // Create HttpResponse and ContentType
        HttpResponse::Ok()
            .content_type(ContentType::json())
            .body(res_body)
    }
}

#[derive(Debug, Serialize)]
struct ErrNoId {
    id: String,
    err: String,
}

// Implement ResponseError for ErrNoId
impl ResponseError for ErrNoId {
    fn status_code(&self) -> StatusCode {
        StatusCode::NOT_FOUND
    }

    fn error_response(&self) -> HttpResponse<BoxBody> {
        let body = serde_json::to_string(&self).unwrap();
        let res = HttpResponse::new(self.status_code());
        res.set_body(BoxBody::new(body))
    }
}

// Implement Display for ErrNoId
impl Display for ErrNoId {
    fn fmt(&self, f: &mut Formatter<'_>) -> std::fmt::Result {
        write!(f, "{:?}", self)
    }
}

// Create AppState
struct AppState {
    books: Arc<Mutex<Vec<Book>>>,
}

// Handler Create Book
#[post("/books")]
async fn post_book(req: web::Json<Book>, data: web::Data<AppState>) -> impl Responder {
    let new_book = Book {
        id: String::from(&req.id),
        title: String::from(&req.title),
        description: String::from(&req.description),
    };
    let response = serde_json::to_string(&new_book).unwrap();
    let mut books = data.books.lock().unwrap();
    books.push(new_book);
    HttpResponse::Created()
        .content_type(ContentType::json())
        .body(response)
}

// handler get all books
#[get("/books")]
async fn get_books(data: web::Data<AppState>) -> impl Responder {
    let books = data.books.lock().unwrap();
    let response = serde_json::to_string(&(*books)).unwrap();
    HttpResponse::Ok()
        .content_type(ContentType::json())
        .body(response)
}

// Handler get a book with the corresponding id
#[get("/books/{id}")]
async fn get_book(id: web::Path<String>, data: web::Data<AppState>) -> impl Responder {
    let book_id: String = id.into_inner();
    let books = data.books.lock().unwrap();
    let book_data = books.iter().find(|x| x.id == book_id);

    match book_data {
        None => {
            let response = ErrNoId {
                id: book_id,
                err: String::from("Book Not Found"),
            };
            Err(response)
        }
        Some(book) => {
            let response = serde_json::to_string(book).unwrap();
            Ok(HttpResponse::Ok()
                .content_type(ContentType::json())
                .body(response))
        }
    }
}

// Handler update the books with the corresponding id
#[put("/books/{id}")]
async fn update_book(
    id: web::Path<String>,
    req: web::Json<Book>,
    data: web::Data<AppState>,
) -> impl Responder {
    let book_id: String = id.into_inner();
    let new_book = Book {
        id: String::from(&req.id),
        title: String::from(&req.title),
        description: String::from(&req.description),
    };
    let mut books = data.books.lock().unwrap();
    let id_index = books.iter().position(|x| x.id == book_id);
    match id_index {
        None => {
            let response = ErrNoId {
                id: book_id,
                err: String::from("Book not found"),
            };
            Err(response)
        }
        Some(id) => {
            let response = serde_json::to_string(&new_book).unwrap();
            books[id] = new_book;
            Ok(HttpResponse::Ok()
                .content_type(ContentType::json())
                .body(response))
        }
    }
}

// Handler delete book with the corresponding id
#[delete("/books/{id}")]
async fn delete_book(id: web::Path<String>, data: web::Data<AppState>) -> impl Responder {
    let book_id = id.into_inner();
    let mut books = data.books.lock().unwrap();
    let id_index = books.iter().position(|x| x.id == book_id);
    match id_index {
        None => {
            let response = ErrNoId {
                id: book_id,
                err: String::from("Book Not Found"),
            };
            Err(response)
        }
        Some(id) => {
            let delete_book = books.remove(id);
            Ok(delete_book)
        }
    }
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let app_state = web::Data::new(AppState{
        books: Arc::new(Mutex::new(vec![
            Book {
                id: String::from("A0001"),
                title: String::from("Atomic Habits"),
                description: String::from("Small changes give extraordinary results.")
            },
            Book {
                id: String::from("A0002"),
                title: String::from("The Psychology of Money"),
                description: String::from("Managing money well has less to do with your intelligence and more to do with your behavior.")
            }
        ]))
    });
    HttpServer::new(move || {
        App::new()
            .app_data(app_state.clone())
            .service(post_book)
            .service(get_books)
            .service(get_book)
            .service(update_book)
            .service(delete_book)
    })
    .bind(("127.0.0.1", 8000))?
    .run()
    .await
}
```

## Run the Application
To run the application, execute the following command in the terminal:
```bash
cargo run
```
This command compiles the Rust code and starts the Actix Web server on `localhost:8000`.

