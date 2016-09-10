---
layout: post
title: "从 CGI、FastCGI、WSGI 角度来了解 Web 的发展"
description: ""
category: web
---

Web 技术发展这么多年，在用户看来只有两部分 HTTP request 和 HTTP response，即：发送请求和返回结果。那从服务器角度观察，在简单的 HTTP request 和 HTTP response 之间又进行了如何的处理实现？

下面就对几种常见的规范进行介绍，分别是：CGI，FastCGI 和 WSGI。

# Overview

## CGI

在 CGI 诞生之前，用户通过 HTTP 请求数据，Web server 会直接从对应路径中查找文件。例如：Web server 设置默认访问路径为 `/usr/local/apache/htdocs/` 那么 `http://xxx.com/htdocs/index.html` 就会对应到服务器中 `/usr/local/apache/htdocs/index.html`文件。

但这存在一个严重的问题问题：用户只能访问静态内容，服务器接收到用户请求后，直接返回对应路径的静态文件，所有人 看到内容都是一样的，但用户定制化的要求越来越高，所以这会带来很大不便。

**CGI 是一个用于在 Web 页面上生成动态内容的一种通用方法。**

> Wikipedia 的定义：Common Gateway Interface (CGI) is a standard method used to generate dynamic content on Web pages and Web applications.

通过在 Web service 和生成动态内容的程序之间提供接口，这样当服务器接收到用户发出的 HTTP 请求后，执行 CGI 程序，CGI 程序通过解析 url 并生成结果，随后 Web service 再将结果通过 HTTP Response 返回。

这样对于网站开发者来说 url 的定制性会更高，对于用户来说也更友好，于是现在的网站会经常看到类似 `/books/32324`, `/books/34244` 这样的路径。同时也对 HTTP 的 `GET`, `POST` 等方法也提供了支持。

## FastCGI

由于 CGI 在处理一个请求时候需要单独起一个进程，这样当请求量较大时，Web server 会运行很多进程，造成较大开销。

**FastCGI 使用固定的进程响应 HTTP 请求，然后将当前环境变量和请求传递给 FastCGI 进程处理，然后将处理结果传递给 Web server，随后 Web server 开始处理请求，把生成的页面数据返回给用户。**

> Instead of creating a new process for each request, FastCGI uses persistent processes to handle a series of requests. These processes are owned by the FastCGI server, not the web server.

> To service an incoming request, the web server sends environment information and the page request itself to a FastCGI process over a socket (in the case of local FastCGI processes on the web server) or TCP connection (for remote FastCGI processes in a server farm). Responses are returned from the process to the web server over the same connection, and the web server subsequently delivers that response to the end-user. The connection may be closed at the end of a response, but both the
web server and the FastCGI service processes persist.

通过将请求接受和请求处理两部分分离，使服务器能用更少的资源响应更多的请求。同时支持更多方式进行请求的处理（例如：单个进程上处理多个请求，多路复用等等），从而增加程序的稳定性和可拓展性。

## WSGI

在 CGI 的基础上，不同语言有各自不同的规范和改进。

**WSGI 是用 Python 实现的 Web server 和Web application/framework 之间定义一个简单而通用的接口。**

WSGI 分为两部分：服务器和应用框架。在处理一个 WSGI 请求时，Web server 会提供环境变量和回调函数给应用程序，应用程序对请求进行处理，处理完成后将结果再返还给回调函数。同时 WSGI 还在 WSGI server 和 WSGI application 之前提供了 WSGI middleware 支持，可以接收到请求后在应用程序处理之前，进行执行一些具体操作。

# 参考资料：

* [CGI](http://en.wikipedia.org/wiki/Common_Gateway_Interface)
* [FastCGI](http://en.wikipedia.org/wiki/FastCGI)
* [WSGI](http://en.wikipedia.org/wiki/Web_Server_Gateway_Interface)
* [PEP 333 - Python Web Server Gateway Interface v1.0](https://www.python.org/dev/peps/pep-0333/)
