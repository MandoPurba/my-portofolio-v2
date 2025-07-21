---
layout: ../../layouts/post.astro
title: "This is the first post of my new Astro blog."
pubDate: 2023-12-23
description: "This is the first post of my new Astro blog."
author: "nicdun"
isPinned: true
excerpt: Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et justo duo dolores et ea rebum. Stet clita kasd gubergren, no sea takimata sanctus est Lorem ipsum dolor sit amet. Lorem ipsum dolor sit amet, consetetur sadipscing elitr, sed diam nonumy eirmod tempor invidunt ut labore et dolore magna aliquyam erat, sed diam voluptua. At vero eos et accusam et
image:
  src:
  alt:
tags: ["tag4", "tag1", "tag2", "tag3"]
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
