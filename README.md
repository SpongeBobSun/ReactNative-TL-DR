# ReactNative - Too Long Don't Read

After using `ReactNative` in company projects for about one year, I've decide to write something about it. There are a lot of documents talking about pros and cons of `ReactNative` so l will not do that. Summarize in one word - `ReactNative` is awesome.

I'll write down the important part and bread crumbs about how `ReactNative` works in this document. Starting with `AppDelegate` of a simple project and we will go through the initialization part of `ReactNative` ,  `NativeModules` and how JavaScript code and native code connected. Along with how does `ReactNative` using `JavaScriptCore`, how does a native UI component get drew and other interesting parts.

The reason why this document is called 'Too Long Don't Read' is I will summarize contents in a chapter to a figure or a bullet list. Then I'll put it on the beginning of each chapter entitled `TL;DR`.

![](/assets/tldr.png)

The `TL;DR` part will provide a quick reference if you want to go through chapters quickly.

## Who this document is for.

This document is for those who already played with `ReactNative`, or made their own app using `ReactNative`  and are curious about:

* How does ReactNative work?
* What's under the hood?

Also this document is based on `ReactNative` version '0.52.0' and **iOS **platform. Although some code written in cpp are used by both iOS and Android but I will **not **go through any Java code.

## Who this document is not for.

If you are a Android developer and wish to read about plenty Java code in `ReactNative` - this is not for you, **currently**. I've planed to add Android part in this document but I will focus on iOS for now.

If you want to learn how to make apps using `ReactNative` - this is not your cup of tea. I'll assume readers have a solid \( well not that solid maybe \) background of using `ReactNative` to make apps.

## How this document is organized.

## Conventions.



