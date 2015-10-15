#响应式处理数据流

Progressive Stream Processing and manipulation is an important task in modern Web Programming, starting from chunked upload/download to Live Data Streams consumption, creation, composition and publishing through different technologies including Comet and WebSockets.

Iteratees provide a paradigm and an API allowing this manipulation, while focusing on several important aspects:

* Allowing the user to create, consume and transform streams of data.
* Treating different data sources in the same manner (Files on disk, Websockets, Chunked Http, Data Upload, …).
* Composable: using a rich set of adapters and transformers to change the shape of the source or the consumer - construct your own or start with primitives.
* Being able to stop data being sent mid-way through, and being informed when source is done sending data.
* Non blocking, reactive and allowing control over resource consumption (Thread, Memory)