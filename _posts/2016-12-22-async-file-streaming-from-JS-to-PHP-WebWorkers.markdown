---
layout: post
title:  "Async file streaming from JS to PHP using WebWorkers"
date:   2016-12-21 23:00:01 +0800
categories: [Angular2, PHP]
comments: true
---

This gist demonstrate how to read a file from the browser using javascript asynchronously with web workers, and how to process the file on PHP server side.

## Angular2 Client Side
For demo and source code check src/app.ts in [Pluker Preview](https://plnkr.co/edit/tAX33FwoQj6YuPmxH6oF?p=preview)

### HTML
Angular2 Template
{% highlight html %}
<input type='file' (change)="openFile($event)">
{% endhighlight %}

A simple input field, that triggers openFile function when on change event fires.

### Typescript
{% highlight javascript linenos %}
openFile(e) {
  if (e.target.files[0]) {
    let file = e.target.files[0];

    // Build a worker from an anonymous function body
    var blobURL = URL.createObjectURL(
      new Blob([
        '(',
        function () {
          self.addEventListener(
            'message',
            function (e) {
              try {
                postMessage({
                  result: new FileReaderSync().readAsArrayBuffer(e.data)
                });
              } catch (e) {
                postMessage({result: 'error'});
              }
            },
          false
          );
        }.toString(),
        ')()' ]
        , {type: 'application/javascript'}
      )
    );

    let worker = new Worker(blobURL);

    worker.onmessage = (e) => {
      let fileData = new Int8Array(e.data.result);
      // fileData contains the streamed file
      this.ngZone.run(() => {
        this.stream = fileData.join(",");
      });
    };

    worker.postMessage(file);
  }
}
{% endhighlight %}

- WebWorkers need to be defined as a seperate JS file, instead we use a Blob to inline the worker from an anonymous function.
- Since webworkers run on their own thread, we have to call `ngZone.run` to update Angular2 view from `syncWorker.onmessage` callback.

### Typings
If your IDE or Webpack complains about missing typings for the webworkers or FileReaderSync functions, add custom typings to your angular2 bootstraping file.

main.bowser.d.ts
{% highlight typescript %}
interface FileReaderSync {
  readAsArrayBuffer(blob: Blob): any;
  readAsBinaryString(blob: Blob): void;
  readAsDataURL(blob: Blob): string;
  readAsText(blob: Blob, encoding?: string): string;
}

declare var FileReaderSync: {
  prototype: FileReaderSync;
  new(): FileReaderSync;
};

declare function postMessage(data: any): void;
{% endhighlight %}

main.browser.ts
{% highlight typescript %}
// <reference path="./main.browser.d.ts" />
{% endhighlight %}

## PHP Server Side
Streamed file can be sent to a PHP Backend using WebSocket and processed in PHP.
Here an example of how to read the Int8Array buffer view in PHP as an array of lines (useful for CSV parsing).

{% highlight php %}
$text = implode(array_map('chr', $file));
$lines = array_filter(explode(PHP_EOL, $text));
{% endhighlight %}
