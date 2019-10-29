#Review code:

![Examplate pipeline!](./images/figure-1.png "The elements are GStreamer's basic construction blocks")

> Element creation

```
/* Create the elements */
source = gst_element_factory_make ("videotestsrc", "source");
sink = gst_element_factory_make ("autovideosink", "sink");
```

New elements can be created with ```gst_element_factory_make()```. The first parameter is the type of element to create. The second parameter is the name we want to give to this particular instance. Naming your elements is useful to retrieve them later if you didn't keep a pointer (and for more meaningful debug output). If you pass NULL for name, howerver, GStreamer will provide a unique name for you.

In this note, two elements are created: a `videotestsrc` and an `autovideosink`

`videotestsrc` is a source element (produce data), which creates a test video pattern.

`autovideosink` is sink element (consume data), which displays on a window the images it receives. There exist serveral video sinks, depending on the operating system, with a varying range of capabilities. `autovideosink` automatically selects and instantiates the best one, so you do not have to worry with the details, and your code is more platform independent.

> Pipeline creation

```
/* Create the empty pipeline */
pipeline = gst_pipeline_new ("test-pipeline");
```

All elements in GStreamer must typically be contained inside a pipeline before they can be used, because it takes care of some clocking and messaging functions. We create the pipeline with ```gst_pipeline_new()```.

```
/* Build the pipeline */
gst_bin_add_many (GST_BIN (pipeline), source, sink, NULL);
if (gst_element_link (source, sink) != TRUE) {
  g_printerr ("Elements could not be linked.\n");
  gst_object_unref (pipeline);
  return -1;
}
```
