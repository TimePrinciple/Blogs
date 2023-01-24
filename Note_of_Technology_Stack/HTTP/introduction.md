# Hyper Text Transfer Protocol

Abbreviated for HTTP. It is the communication protocol between *web servers* and *clients*.

## HTTP is stateless

- Every request is completely **independant**. It doesn't remember about previously transaction, each request is a single transaction.
- Programming, Local Storage, Cookies, Sessions are used to create enhanced user experiences.

## Hyper Text Transfer Protocol Secure

Abbreviated for HTTPS. All the data that sent back and forth are encrypted by Secure Sockets Layer (SSL) or by Transport Layer Security (TLS). *Any time users are sending sensitive information, should always be over HTTPS, especially like credit card data, social security number, where high level security is needed.* **Forcing HTTPS on every page is a bad idea.**

## Methods

### GET

Retrieve data from the server. Every time visiting a page, a GET request to the server is been madevia HTTP.

e.g. Loading a standard HTML page; loading assets like CSS, images, JSON data, XML data and so on.

### POST

Submit (add) data to the server. The data been sent to the server will be stored in its database or somewhere. **A form can also be sent through a GET request, but the content in the form is actually visible int he URL, which is less secure.** Typically GET request won't be used unless it's a search form or filtering data comming back from the server (which are not actually post anything).

e.g. Submit a form, a blog post.

### PUT

Update data already on the server.

e.g. Editing an exisiting blog post.

### DELETE

Delete data from the server.

## Header files.

With each request and response, there are header and body. The body typically in a response is the HTML page which is been loading, JSON data or whatever is being sent from the server. The body in a request are the fields in a submitting form.
