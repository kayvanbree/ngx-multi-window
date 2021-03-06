# ngx-multi-window [![npm version](https://img.shields.io/npm/v/ngx-multi-window.svg?style=flat)](https://www.npmjs.com/package/ngx-multi-window) [![MIT license](http://img.shields.io/badge/license-MIT-brightgreen.svg)](http://opensource.org/licenses/MIT)

Pull-based cross-window communication for multi-window angular applications

[![Build Status](https://travis-ci.org/Nolanus/ngx-multi-window.svg?branch=master)](https://travis-ci.org/Nolanus/ngx-multi-window)
[![Dependency Status](https://david-dm.org/Nolanus/ngx-multi-window.svg)](https://david-dm.org/Nolanus/ngx-multi-window)
[![devDependency Status](https://david-dm.org/Nolanus/ngx-multi-window/dev-status.svg)](https://david-dm.org/Nolanus/ngx-multi-window?type=dev)
[![peerDependency Status](https://david-dm.org/Nolanus/ngx-multi-window/peer-status.svg)](https://david-dm.org/Nolanus/ngx-multi-window?type=peer)

[![Greenkeeper badge](https://badges.greenkeeper.io/Nolanus/ngx-multi-window.svg)](https://greenkeeper.io/)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat)](http://makeapullrequest.com)
[![Codacy Badge](https://api.codacy.com/project/badge/Grade/b175dcd8585a42bdbdb9c1ee2a313b3b)](https://www.codacy.com/app/sebastian-fuss/ngx-multi-window?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=Nolanus/ngx-multi-window&amp;utm_campaign=Badge_Grade)

## Features

- Send messages between different tabs/windows that are running the angular app
- Message receive notification for sending tab/window
- Automatic detection/registration of new tabs/windows

## Setup

First you need to install the npm module:
```sh
npm install ngx-multi-window --save
```

Then add the `MultiWindowModule` to the imports array of your application module:

```typescript
import {MultiWindowModule} from 'ngx-multi-window';

@NgModule({
    imports: [
        /* Other imports here */
        MultiWindowModule
        ]
})
export class AppModule {
}
```

Finally you need to specify how your application should load the ngx-multi-window library:

### Angular modules

All the compiled JavaScript files use ES2015 module format, so they are ready for usage with [RollupJS](http://rollupjs.org/). However, you cannot use them with SystemJS.

`.metadata.json` files are generated for usage with [Angular AoT compiler](https://angular.io/docs/ts/latest/cookbook/aot-compiler.html).

### SystemJS

UMD bundles are available for SystemJS loading. Example:

```js
System.config({
    paths: {
        'npm:': 'node_modules/'
    },
    map: {
        app: 'app',

        '@angular/core'   : 'npm:@angular/core/bundles/core.umd.js',
        '@angular/common' : 'npm:@angular/common/bundles/common.umd.js',
        // further angular bundles...

        'ngx-multi-window': 'npm:ngx-multi-window/bundles/ngx-multi-window.umd.js',

        rxjs: 'npm:rxjs',
    },
    packages: {
        app : {defaultExtension: 'js', main: './main.js'},
        rxjs: {defaultExtension: 'js'}
    }
});
```
## Usage

Inject the `MultiWindowService` into your component or service.

```typescript
import {MultiWindowService} from 'ngx-multi-window';

export class AppComponent {
    constructor(private multiWindowService: MultiWindowService) {
        // use the service 
    }
}   
```

### Window ID and name

Every window has a unique, unchangeable id which can be accessed via `multiWindowService.id`.
In addition to that every window as a changeable name which can be get/set 
via `multiWindowService.name`.

### Receive messages

Receive messages addressed to the current window by subscribing to the observable returned from
`multiWindowService.onMessages()`:

```typescript
import { MultiWindowService, Message } from 'ngx-multi-window';

class App {
    constructor(private multiWindowService: MultiWindowService) {
        multiWindowService.onMessage().subscribe((value: Message) => {
          console.log('Received a message from ' + value.senderId + ': ' + value.data);
        });
    } 
}
```

### Send messages

Send a message by calling `multiWindowService.sendMessage()`:

```typescript
import { MultiWindowService, WindowData, Message } from 'ngx-multi-window';

class App {
    constructor(private multiWindowService: MultiWindowService) {
        const recipientId: string; // TODO
        const message: string; // TODO
        multiWindowService.sendMessage(recipientId, 'customEvent', message).subscribe(
          (messageId: string) => {
            console.log('Message send, ID is ' + messageId);
          },
          (error) => {
            console.log('Message sending failed, error: ' + error);
          },
          () => {
            console.log('Message successfully delivered');
          });
    }
}
```
The message returns an observable which will resolve with a message id once the message has been send (= written to local storage).
The receiving window will retrieve the message and respond with a `MessageType.MESSAGE_RECEIVED` typed message. 
The sending window/app will be informed by finishing the observable.

In case no `MessageType.MESSAGE_RECEIVED` message has been received by the sending window 
within a certain time limit (`MultiWindowConfig.messageTimeout`, default is 10s) 
the message submission will be canceled. The observable will be rejected and the 
initial message will be removed from the current windows postbox. 

### Other windows

To get the names and ids of other window/app instances the `MultiWindowService` offers two methods:

`multiWindowService.onWindows()` returns an observable to subscribe to in case you require periodic updates of the 
fellow windows. The observable will emit a new value every time the local storage has been scanned fpr the windows. 
This by default happens every 5 seconds (`MultiWindowConfig.newWindowScan`).

Use `multiWindowService.getKnownWindows` to return an array of `WindowData`.

### New windows

No special handling is necessary to open new windows. Every new window/app will register itself 
by writing to its key in the local storage. Existing windows will identify new windows 
after `MultiWindowConfig.newWindowScan` at the latest.

The `MultiWindowService` offers convenient method `newWindow()` which provides details for the 
new window's start url. If used the returned observable can be utilized to get notified 
once the new window is ready to consume/receive message. 

## Communication strategy

This library is based on "pull-based" communication. Every window periodically checks the local storage for messages addressed to itself.
For that reason every window has its own key in the local storage, whose contents/value looks like:

```json
{"heartbeat":1508936270103,"id":"oufi90mui5u5n","name":"AppWindow oufi90mui5u5n","messages":[]}
```

The heartbeat is updated every time the window performed a reading check on the other window's local storage keys.

Sending message from sender A to recipient B involves the following steps:
- The sender A writes the initial message (including the text and recipient id of B) into the "messages" array located at its own local storage key
- The recipient window B reads the messages array of the other windows and picks up a message addressed to itself
- B places a "MESSAGE_RECEIVED" message addressed to A in its own messages array
- A picks up the "MESSAGE_RECEIVED" message in B's message array and removes the initial message from its own messages array
- B identifies that the initial message has been removed from A's messages array and removes the receipt message from its own messages array

## Example App

The [_demo_](demo) subfolder contains a project created with angular-cli
that has been adapted to showcase the functionality of ngx-multi-window. Run the 
demo app by checking out that repository and execute the 
following command in the project root directory:

 ```
 npm run demo
 ```
  
This will perform the following steps:

* Install the dependencies of the ngx-page-scroll project
* Install the dependencies of the demo-app project
* Replace the ngx-page-scroll library in the demo-app with a fresh build
* Run the demo server (`ng serve`)
 
## TODO

- Tests and cross browser testing

## License

[MIT](LICENSE)
