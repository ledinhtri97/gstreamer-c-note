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