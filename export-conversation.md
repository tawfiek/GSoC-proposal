# Export Conversations Functionality

## Introduction

Ability to export conversations is one of the most important features in communication Apps in general, making the user able to backup his history and saving it into his own local machine is one of the key features that most communications Apps provide out there.

As this feature is very important but it also seams very simple to implement as one two three, right ?

1. GET the messages from the server.
2. FORMAT the data into the any desired format.
3. DOWNLOAD the file.

Well, it is not that easy. as Matrix is end to end encrypted so the "messages" here is not a plain text.

This might add some complexity and limitation to those simple three steps above that will be explained in this next sections. but for now lets map our mind around those three simple steps GET, FORMAT then, DOWNLOAD.


## Matrix Architecture

First things first, we need understand how matrix runs. I'll cover briefly my understanding about the most involved parts in this project.

Matrix client are consists of three main layers:

![Image 1](./assets/1.png)

### Matrix JS SDK
>This SDK provides a full object model around the Matrix Client-Server API and emits events for incoming data and state changes. Aside from wrapping the HTTP API.

`matrix-js-sdk` provides a lot of APIs that might be useful to solve our problem such as:

* Handles Pagination
* Handles syncing (via /initialSync and /events)
* Handles historical `RoomMember` information (e.g. display names)
* Handle queueing of messages.

And more.

### Matrix React SDK
> This is a react-based SDK for inserting a Matrix chat/voip client into a web page.

In this layer we will add all web and all react components needed

