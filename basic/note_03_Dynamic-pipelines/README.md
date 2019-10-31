#Review code:

```
/* Structure to contain all our information, so we can pass it to callbacks */
typedef struct _CustomData {
  GstElement *pipeline;
  GstElement *source;
  GstElement *convert;
  GstElement *sink;
} CustomData;
```

Group all data in a struct for easier handling.

```
/* Handler for the pad-added signal */
static void pad_added_handler (GstElement *src, GstPad *pad, CustomData *data);
```

This is a forward reference, to be used later.

```
/* Create the elements */
data.source = gst_element_factory_make ("uridecodebin", "source");
data.convert = gst_element_factory_make ("audioconvert", "convert");
data.sink = gst_element_factory_make ("autoaudiosink", "sink");
```
```uridecodebin``` will internally instantiate all the necessary elements (source, demuxers and decoders) to turn a URI into raw audio and/or video streams. It does half the work that ```playbin``` does. Since it contains demuxers, its source pads are not initally available and we will need to link to them on the fly.

```audioconvert``` is useful for converting between different audio formats, making sure that this example will work on any platform, since the format produced by the audio decoder might not be the same that the audio sink expects.

The ```autoaudiosink``` is the equivalent of ```autovideosink``` seen in the previous note, for audio. It will render the audio stream to the audio card.

```
if (!gst_element_link (data.convert, data.sink)) {
  g_printerr ("Elements could not be linked.\n");
  gst_object_unref (data.pipeline);
  return -1;
}
```

Here we link the converter element to the sink, but we DO NOT link them with the source, since at this point it contains no source pads. We just leave this branch (converter+sink) unlinked, until later on.

```
/* Set the URI to play */
g_object_set (data.source, "uri", "https://www.freedesktop.org/software/gstreamer-sdk/data/media/sintel_trailer-480p.webm", NULL);
```

We set the URI of the file to play via a property, just like we did in the previous note.

```
/* Connect to the pad-added signal */
g_signal_connect (data.source, "pad-added", G_CALLBACK (pad_added_handler), &data);
```
```

```GSignals``` are a crucial point in GStreamer. They allow you to be notified (by means of a callback) when something interesting has happened. Signals are identified by a name, and each ```GObject``` has its own signals.

In this line, we are <i>attaching</i> to the "pad-added" signal of out source (an ```uridecodebin``` element). To do so, we use ```g_signal_connect()``` and provide the callback function to be used (```pad_added_handler```) and a data pointer. GStreamer does nothing with this data pointer, it just forwards it to the callback so we can share information with it. In this case, we pass a pointer to the ```CustomData``` structure we built specially for this purpose.

The signals that ```GstElement``` generates can be found in its documentation or using the ```gst-inspect-1.0``` tool (to find out how, let google it :D).

We are now ready to go! Just set the pipeline to the ```PLAYING``` state and start listening to the bus for interesting messages (like ```ERROR``` or ```EOS```), just like in the previous tutorials.


> The callback

When our source element finally has enough information to start producing data, it will create source pads, and trigger the "pad-added" signal. At this point our callback will be called:

```
static void pad_added_handler (GstElement *src, GstPad *new_pad, CustomData *data) {}
```

```src``` is the ```GstElement``` which triggered the signal. In this example, it can only be the ```uridecodebin```, since it is the only signal to which we have attched. The first parameter of a signal handler is always the object that has triggered it.

```new_pad``` is ```GstPad``` that has just been added to the ```src``` element. This is usually the pad to which we want to link.

```data`` is the pointer we provided when attaching to the signal. In this example, we use it to pass the ```CustomData``` pointer.

```
GstPad *sink_pad = gst_element_get_static_pad (data->convert, "sink");
```

From ```CustomData``` we extract the converter element, and then retrieve its sink pad using ```gst_element_get_static_pad()```. This is the pad to which we want to link ```new_pad```. In the previous tutorial we linked element against element, and let GStreamer choose the appropriate pads. Now we are going to link the pads directly.

```
/* If our converter is already linked, we have nothing to do here */
if (gst_pad_is_linked (sink_pad)) {
  g_print ("We are already linked. Ignoring.\n");
  goto exit;
}
```

```uridecodebin``` can create as many pads as it sees fit, and for each one, this callback will be called. These lines of code will prevent us from trying to link to a new pad once we are already linked.

```
/* Check the new pad's type */
new_pad_caps = gst_pad_get_current_caps (new_pad, NULL);
new_pad_struct = gst_caps_get_structure (new_pad_caps, 0);
new_pad_type = gst_structure_get_name (new_pad_struct);
if (!g_str_has_prefix (new_pad_type, "audio/x-raw")) {
  g_print ("It has type '%s' which is not raw audio. Ignoring.\n", new_pad_type);
  goto exit;
}
```

Now we will check the type of data this new pad is going to output, because we are only interested in pads producing audio. We have previously created a piece of pipeline which deals with audio (an ```audioconvert``` linked with an ```autoaudiosink```), and we will not be able to link it to a pad producing video, for example.

```gst_pad_get_current_caps()``` retrieves the current capabilities of the pad (that is, the kind of data it currently outputs), wrapped in a ```GstCaps``` structure. All possible caps a pad can support can be queried with ```gst_pad_query_caps()```. A pad can offer many capabilities, and hence ```GstCaps``` can contain many ```GstStructure```, each representing a different capability. The current caps on a pad will always have a single ```GstStructure``` and represent a single media format, or if there are no current caps yet ```NULL``` will be returned.

Since, in this case, we know that the pad we want only had one capability (audio), we retrieve the first ```GstStructure``` with ```gst_caps_get_structure()```.

Finally, with ```gst_structure_get_name()``` we recover the name of the structure, which contains the main description of the format (its media type, actually).

If the name is not ```audio/x-raw```, this is not a decoded audio pad, and we are not interested in it.

Otherwise, attempt the link:

```
/* Attempt the link */
ret = gst_pad_link (new_pad, sink_pad);
if (GST_PAD_LINK_FAILED (ret)) {
  g_print ("Type is '%s' but link failed.\n", new_pad_type);
} else {
  g_print ("Link succeeded (type '%s').\n", new_pad_type);
}
```

```gst_pad_link()``` tries to link two pads. As it was the case with gst_element_link(), the link must be specified from source to sink, and both pads must be owned by elements residing in the same bin (or pipeline).

And we are done! When a pad of the right kind appears, it will be linked to the rest of the audio-processing pipeline and execution will continue until ```ERROR``` or ```EOS```. However, we will squeeze a bit more content from this tutorial by also introducing the concept of State.

> GStreamer States

We already talked a bit about states when we said that playback does not start until you bring the pipeline to the ```PLAYING``` state. We will introduce here the rest of states and their meaning. There are 4 states in GStreamer:

|  State   |      					Description      							|
|----------|:------------------------------------------------------------------:|
| NULL	   | the NULL state or initial state of an element. |
| READY    | the element is ready to go to PAUSED.   |
| PAUSED   | the element is PAUSED, it is ready to accept and process data. Sink elements however only accept one buffer and then block. |
| PLAYING  | the element is PLAYING, the clock is running and the data is flowing.|