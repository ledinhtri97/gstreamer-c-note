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

ledinhtri.com





