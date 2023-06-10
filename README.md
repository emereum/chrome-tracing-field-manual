# Chrome Tracing Field Manual

Are you investigating a performance issue and devtools doesn't quite tell you what you need to know? It may be time to dip into `chrome://tracing`.

The following manual consists of problem-oriented guides to getting the most out of `chrome://tracing` ("when you experience _problem X_ look at _feature Y_ in `chrome://tracing`). Some basic understanding of `chrome://tracing` (hereafter called `tracing`) will be required, and is covered in the next few sections.

## Tracing basic concepts

### How to capture and filter a trace

Close off all other chrome processes and tabs except for the tab you want to trace. `tracing` traces all browser processes and tracing irrelevant tabs needlessly fills up the tracing buffer.

In a new tab, navigate to `chrome://tracing`. At the top left you'll see a `Record` button. Click it, accept all default options, then click `Record`. Now refresh the page you want to trace. You should see the tracing buffer begin to fill up. Click `Stop` and you'll be presented with a trace that looks something like this:

![[Pasted image 20230610171902.png]]

These data need to be filtered to be useful. Click the `Processes` button (top right), then uncheck all but the tab of interest. Look for a name like `Renderer (pid 1234): Your Page Title`.

### How to navigate a trace
#### Look at `CrRendererMain` first
Now that you have a trace and have filtered it down to your tab of interest, let's navigate through it. First scroll down to `CrRendererMain`. You should see some spans (horizontal bars) but they'll be too zoomed out to be meaningful.

> **Note**
> `CrRendererMain` represents the renderer process for a tab. It's the process in which your javascript, html, CSS, etc. run. There may be other relevant processes to look at, such as `CrGpuMain` which represents the GPU process. For more information, read [Background: How Chrome Renders Web Pages](https://www.chromium.org/developers/how-tos/trace-event-profiling-tool/trace-event-reading/#background-how-chrome-renders-web-pages)

#### WASD
`tracing` uses Quake-style navigation. Use the keys `A, D` to navigate back and forward in time. Use the keys `W, S` to zoom in and out. Hold `shift` to zoom faster. Zooming operations are centered around the mouse cursor, so place the cursor on a block of spans and zoom in:

![[Pasted image 20230610172952.png]]

#### Mouse modes: 1, 2, 3, 4
`tracing` also has four mouse modes which can be toggled using the floating mouse mode selector to the right of the window, or with keys `1, 2, 3, 4`:

1: Mouse selection (important)
2: Mouse move
3: Mouse zoom
4: Mouse timing

### How to analyse traces
Put your mouse into selection mode (press `1`) then click and drag to select all the spans in `CrRendererMain`. You should see the analysis view populate with data (at the bottom of the window). Click the `Slices` tab if it's not already selected:

![[Pasted image 20230610173841.png]]

`ThreadControllerImpl::RunTask` should be at the top of the list for `Wall Duration`; this method is the initiator for _any_ renderer activity. If a page is slow and there is no `RunTask` span during the slow period, the problem is either browser or GPU related.

#### View details for a specific trace name
At the moment the analysis view is showing a summary table because we have selected all kinds of traces. Click `ThreadControllerImpl::RunTask` in the table. You'll see more details now that we have focused on one type of trace.

You can use your browser's back and forward buttons to navigate through the analysis view.

![[Pasted image 20230610182457.png]]

The `ThreadControllerImpl::RunTask` traces appear to log some arguments. Different trace types will log different information. Pick one invocation that looks interesting, I chose this one: 

| Args  | |
|---|---|
|src_file|"third_party/blink/renderer/core/html/parser/html_document_parser.cc"|
|src_func|"SchedulePumpTokenizer"|

Now plug that `src_file` into https://source.chromium.org/ then search for the `src_func` value to get more context. It seems `html_document_parser` fires the tracing event [here](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/core/html/parser/html_document_parser.cc;l=783?q=third_party%2Fblink%2Frenderer%2Fcore%2Fhtml%2Fparser%2Fhtml_document_parser):

```
void HTMLDocumentParser::SchedulePumpTokenizer(bool from_finish_append) {
  TRACE_EVENT0("blink", "HTMLDocumentParser::SchedulePumpTokenizer");
  ...
```

which schedules the `HTMLDocumentParser::DeferredPumpTokenizerIfPossible` function here:

```
  ...
  loading_task_runner_->PostDelayedTask(
      FROM_HERE,
      WTF::BindOnce(&HTMLDocumentParser::DeferredPumpTokenizerIfPossible,
                    WrapPersistent(this), from_finish_append,
                    base::TimeTicks::Now()),
      delay);
  ...
```

Sure enough, if we look at the tracing data we do find that `DeferredPumpTokenizerIfPossible` was called by `RunTask` :

![[Pasted image 20230610183938.png]]

What does this mean? Not a whole lot since this is just an exercise to get familiar with `tracing`. It is interesting to note that our trace took us to `chromium/src/third_party/blink`, however.

> **Note**
> Blink does a lot of heavy lifting in Chrome. It processes the DOM, CSS and Javascript among other things. Blink depends on V8 (for running Javascript) and Skia (for rendering graphics). To learn the basics about Blink, see [How Blink works](https://docs.google.com/document/d/1aitSOucL0VHZa9Z2vbRJSyAIsAz24kX8LFByQ5xQnUg/edit)

At this point, you might realise that to understand what your code is doing, you need to understand `tracing`. To understand `tracing` you need to understand how Chrome works. This manual is as much about learning Chrome as it is about `tracing`.

#### Clone the Chromium repository
Since we'll be looking at a lot of Chrom(ium) source code, it's worth cloning the repo. Follow the instructions for your OS [here](https://source.chromium.org/chromium/chromium/src/+/main:docs/README.md)

We're now armed with enough knowledge to do something useful with `tracing`. The next sections dive straight into problem-focused examples (called scenarios).

## Tracing scenarios

tbd