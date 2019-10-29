#Review code:

```
/*Initialize GStreamer*/
gst_init(&argc, &argv);
```

Alway be the first GStreamer command. gst_init() do:

> - Initializes all internal structures
> - Checks what plug-ins are available
> - Executes any command-line option intended for GStreamer

parameters ```args``` and ```argv``` to gst_init() will automatically benefit from the GStreamer standard command-line options.

```
/*Build the pipeline*/
pipeline = gst_parse_launch
	("playbin uri=https://www.freedesktop.org/software/gstreamer-sdk/data/media/sintel_trailer-480p.webm",
	NULL);
```

> gst_parse_launch

GStreamer allows media traveling from the "source" elements (the producers), down to the "sink" elements (the consumers). The set of all the interconnected elements is called a "pipeline".

When pipeline is easy enough, and do not need any advanced features, you can take the shortcut: ```gst_parse_launch()```

> playbin

Playbin is the special element which acts as a source and as a sink, and is a whole pipeline.

```
/*Start playing*/
gst_element_set_state(pipeline, GST_STATE_PLAYING)

```

Every GStreamer element has an associated state, which you can more or less think of as the Play/Pause button in your regular DVD player.

```
/* Wait until error or EOS */
bus = gst_element_get_bus (pipeline);
msg =
    gst_bus_timed_pop_filtered (bus, GST_CLOCK_TIME_NONE,
    GST_MESSAGE_ERROR | GST_MESSAGE_EOS);
```

These lines will wait until an error occurs or the end of the stream is found. ```gst_element_get_bus()``` retrieves the pipeline's bus, and ```get_bus_timed_pop_filtered()``` will block until you receive either an ERROR or an ```EOS``` (End-of-Stream) through that bus.

Execution will end when the media reaches its end (EOS) or an error is encountered (try closing the video window or unplugging the network cable). The application can always be stopped by pressing control-C in the console.

> Cleanup

Before terminating the application, though, there is a couple of things we need to do tidy up correclty after ourselves.

```
/* Free resources */
if (msg != NULL)
  gst_message_unref (msg);
gst_object_unref (bus);
gst_element_set_state (pipeline, GST_STATE_NULL);
gst_object_unref (pipeline);
```

Should free the objects they return after using them.

In this case, gst_bus_timed_pop_filtered() returned a message which needs to be freed with gst_message_unref()

gst_element_get_bus() added a reference to the bus that must be freed with gst_object_unref(). Setting the pipeline to the NULL state will make sure it frees any resources it has allocated, Finally, unreferencing the pipeline will destroy it, and all its contents.

#What we have learned:

> - How to initialize GStreamer using gst_init()
> - How to quickly build a pipeline from a textual description using gst_parse_launch().
> - How to create an automatic playback pipeline using playbin.
> - How to signal GStreamer to start playback using gst_element_set_state()
> - How to sit back and relax, while GStreamer takes care of everything, using gst_element_get_bus() and gst_bus_timed_pop_filtered().