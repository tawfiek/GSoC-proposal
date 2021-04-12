# Personal Information

**Name:** Tawfiek allah Khalaf Eldeep

**Phone:** +201118772996

**E-mail:** tawfiek.108@gmail.com

**LinkedIn:**  [in/tawfiek](https://www.linkedin.com/in/tawfiek/)

**Twitter:**  [@tawfiekallah](https://twitter.com/tawfiekallah)



# About me

## Introduction
Software developer with 4 years of experience in the market, having good experience with open source communities.
It is my first contribution with Matrix this year as a GSoC participants, currently I am working in one of fin-tech startups companies as a full stack software developer.

Also I am currently studying communication and electronics engineering part-time.

## Open Source Contributions

I consider myself as an open source lover, I believe in open source value to the development community and improving my development skills, that's why I am trying to contribute from time to time into one of projects that I use everyday in my work, one of the most contribution I am proud of that one with ionic framework [#16851](https://github.com/ionic-team/ionic-framework/pull/16851), this one I was need to implement a `text-area` in ionic alert component in one of projects I was working on, and there was no way to do that in ionic 4 ( the version I was using this time), I thought okay lets go impalement it myself in ionic , why not ?!

As it is my final year in the collage, I think I'll have more time in the future to pay more attention to the Open source in general. and I think GSoC and Matrix will be a good station to help me to do that.

## My contributions with matrix

I didn't do too much with `Matrix` Actually in the last period I tried to  make my hands dirty in some easy issues in `element` I did make some pull requests on `matrix-react-sdk` solving some stuff, also I got familiar with `hydrogen` and it's SSO feature request, I think it was great journey to get into different matrix solutions code base.

### Matrix React SDK

* [Make buttons in verify dialog respect the system font ](https://github.com/matrix-org/matrix-react-sdk/pull/5778)
* [Adding description for Clear Cache and Reload Button in Settings](https://github.com/matrix-org/matrix-react-sdk/pull/5811)


### Hydrogen

* [Setup docdash for the project](https://github.com/vector-im/hydrogen-web/pull/302)
* [Add SSO functionality](https://github.com/vector-im/hydrogen-web/pull/282)


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

## Implementing Exporting Functionality

> NOTE: All codes in this document is just my thoughts about the implementation, The purpose of writing this code is rounding out the idea and simplifying it for the reader, and it is not considered as a production ready code.  

As discussed with the community [here](https://github.com/vector-im/element-web/issues/2630).
The functionality should be implemented on each room individually, and also we have to add a way to limit the number of exported events, that because of the expenses of decrypting on the client side ( Resources limitations )  

As mentioned in introduction this function has three phases to implement GET, FORMAT and DOWNLOAD.  
In this section I'll describe How I thought about implementing those there phases.
### GET

As `matrix-js-sdk` holds all communications with client server and handles all events and encryption stuff, I think it is the best place to add this phase into this SKD let it handle getting the room's events ( messages and state events ) by calling  [ GET /_matrix/client/r0/rooms/{roomId}/messages](https://matrix.org/docs/spec/client_server/latest#id267)  from the home server getting encrypted events from the server.
> note: A private function into `MatrixClient` can be used to make this call `_createMessagesRequest`

> All this logic should be added into `MatrixClient` class to be provided in the client object into `matrix-react-sdk`

here is a pseudo code for how we can impalement the GET function into `MatrixClient`

``` javascript
    MatrixClient.prototype._exportMessagesHistoryByRoomID = async function  (roomID, limit) {
        // limit here should be calculated by the consumer
        const token = eventTimeline.getPaginationToken(dir);
        // direction here should be BACKWARDS always.
        const dir = EventTimeline.BACKWARDS;

        const response = await this._createMessagesRequest(
            roomID,
            token,
            limit,
            dir);

        const encryptedEvents = await this._decryptEventsForExport(response);

        return encryptedEvents
    }

```
Well, until this point we have all the events needed to export now lets implement `_decryptEvents()` function that decrypts the events.

Here we have two types of events `StateEvents` and `MatrixEvents`, we can found it into `state` and `chunk` array respectively in the API call response.

we can use  `map` function ( functional programming ) to map the encrypted events arrays into typed objects that represent the `stateEvents` and `matrixEvents` that has all functionality of decrypting them into

now we have ton of events that we need to be decrypted.

 into `MatrixEvents` we have function called `attemptDecryption` that called into the mapper mentioned above and this function sets `_decryptionPromise` private property into the event object, this function also waiting util the promise fulfield and set another private property called `_clearEvent` looks like this

``` javascript
{
    _clearEvent: {
        content:
        body: "Hello their"
        msgtype: "m.text"
    }
}
```

yes this what we waiting for.

we need to make some function into `MatrixEvent` that keeps tracking an array  of events to be decrypted and return all decrypted messages.

> At this point we might fall into recourses issue because the events here must be limited to not exceed the call stack on browser.  

whoever the  `_decryptEventsForExport` should be looks like this

```javascript
MatrixClient.prototype._decryptEventsForExport = async function (res) {
    const stateEvents = res.state ?  utils.map(res.state, self.getEventMapper():{};
    const matrixEvents = utils.map(res.chunk, self.getEventMapper());

    // This function should implemented into MatrixEvent class that takes an array of events
    // and return a promise resolves after all promises of decryption if ended  
    const decryptedMatrixEvents = await MatrixEvent.decryptAll(matrixEvents);
    const decryptedStateEvents = await await MatrixEvent.decryptAll(stateEvents);

    return {
        stateEvents: decryptedStateEvents,
        matrixEvents: decryptedMatrixEvents
    }
}
```

Now as a result we have all events we need decrypted, actually not all of them.

Some events might be not decrypted because of losing megolm sessions needed to decyrpt it in this case we might can use key backup, or request keys from your other devices, after that if we still don't have the keys, then we give up and say we just can't decrypt it.

Okay, now we've done with this phase lets go and format the data we got form this function.

### FORMAT

Formatting the returned event should be very simple task, get the desired format from the user , map on the array and write your file as simple as that.

Matrix has a lot of event types and all types are manged into `matrix-react-sdk` in function called `textForEvent` in `TextForEvent.js` file.

This function takes the matrix event object and identify its type then returns translated string that describe the event, "Magic".

Now we should make a function that write the Blob into text format, pass the Blob to the download function

```javascript

    // Format function should be called with events to format and the desired format
    // For the GSoC project we gonna implement just the text format.
    // And extends the feature over time to support more formats.

    function formatExportedEve (events: MatrixEvent, format: String ): Blob {
        if (format === 'text') {
            const textContent = events.map(textForEvent);

            return new Blob(textContent, {
                type: "text/plain;charset=utf-8"
            });
        } // .. all formats should be implemented in the future
    }
```

### DOWNLOAD
In this phase all data should be arrived to the very starting point in the component, then we can use library like [file saver](https://github.com/eligrey/FileSaver.js) to download, this library is provide us a nice to use mthod to download the file easily

``` javascript
    const content = "What's up , hello world";
    // any kind of extension (.txt,.cpp,.cs,.bat)
    const filename = "hello.txt";

    const blob = new Blob([content], {
        type: "text/plain;charset=utf-8"
    });

    saveAs(blob, filename);
```

But it comes with a little support limitations that we have to handle

Supported Browsers
------------------

| Browser        | Constructs as | Filenames    | Max Blob Size | Dependencies |
| -------------- | ------------- | ------------ | ------------- | ------------ |
| Firefox 20+    | Blob          | Yes          | 800 MiB       | None         |
| Firefox < 20   | data: URI     | No           | n/a           | [Blob.js](https://github.com/eligrey/Blob.js) |
| Chrome         | Blob          | Yes          | [2GB][3]      | None         |
| Chrome for Android | Blob      | Yes          | [RAM/5][3]    | None         |
| Edge           | Blob          | Yes          | ?             | None         |
| IE 10+         | Blob          | Yes          | 600 MiB       | None         |
| Opera 15+      | Blob          | Yes          | 500 MiB       | None         |
| Opera < 15     | data: URI     | No           | n/a           | [Blob.js](https://github.com/eligrey/Blob.js) |
| Safari 6.1+*   | Blob          | No           | ?             | None         |
| Safari < 6     | data: URI     | No           | n/a           | [Blob.js](https://github.com/eligrey/Blob.js) |
| Safari 10.1+   | Blob          | Yes          | n/a           | None         |

> This lib is just a recommendation since I had interacted with it before, we can use any other alternative, or sure we can implement our own file saver if we don't have one already in the codebase.

# Commitment

I plan to work with matrix to solve the above problem from 4 - 6 hours a day expect my exam period.

My exams will take place from 3rd of July until 17th of July, so I won’t be able to work with my full capacity for about 14 days, but I will be there for few hours in the day.
# Schedule

Estimated Duration | Estimated Start/End Time     | Task
-------------------| ---------------------------- | -------------------------------------
1 Month            | April 13, 2021 - May 16, 2021| Continue practicing in matrix in some related area to the project, and communicate with community and mentors to discuss the project
4 weeks            | May 17, 2021 - June 7, 2021  | Community Bonding. in this period should discuss the implementation with the community into the communication channels and the related issue on github for better understanding the limitations and all corner cases should be handled in this feature.
6 weeks            | July 16, 2021 - August 16, 2021 | Working on the GET phase explained above and add all test cases on `matrix-js-sdk`.  <br /> after this period a fully working API should be implemented and tested to get decrypted events from the server and get ready for the next phase. ( The GET phase is most critical phase in this feature since it has all our concerns, thats why I give it this much time to carefully test all use cases ). <br /> At the end of this period I am planing to start with the UI components should be implemented into `matrix-react-sdk`,<br /> that will be the starting point for us to start impalement the remaining two phases in the next coding period.
1 week             | August 16 - 23, 2021         | First Evaluations.
1 month            | July 16, 2021 - August 16, 2021 | Starting with the UI component implemented in the last period, we should impalement the UX and starting adding the logic of other two phases FORMAT and DOWNLOAD and making the wrapper function that will organize the data flow in the app until the download will start.
2 weeks            | August 16 - 30, 2021        | All codes should be submitted and discuss any issues with the community
Ongoing             | -- | Support this feature and enhance it over time to fits our community needs, Extends the number of formats should be available to export conversion.
