# Probes-handling-in-GStreamer-pipelines
This is a brief post explaining the concept of probes and its usage w.r.t GStreamer pipelines


GStreamer provides an excellent concept of adding probes to the pipeline elements, that can be used for a number of purposes- to get notified of upstream/downstream events, push/pull buffers, idle activity, etc. Pad probes are also useful in case of dynamic pipelines, where the pipeline elements are added/delted on the fly. 

A very good post related to GStreamer probes can be found [here](https://coaxion.net/blog/2014/01/gstreamer-dynamic-pipelines/) .
This post, takes inspiration from the mentioned blog, and explains the same implementation w.r.t Python. This post is more focussed on the implementation aspects, particulary the challenges I faced when implementing the GStreamer-probes using Python.

Consider the following pipeline, where I receive video packets via UDP channel:

    udpsrc name=src ! video/x-raw ! videoconvert ! autovideosink

Step 1. Install a probe on the source i.e. `udpsrc`
    
    # get the element handle from pipeline
    src_element = pipeline.get_by_name('src')
    
    #get the static source pad of the element
    srcpad = src_element.get_static_pad('src')
    
    #add the probe to the pad obtained in previous solution
    probeID = srcpad.add_probe(Gst.PadProbeType.BUFFER, probe_callback)
    
 The API [`add_probe()`](https://valadoc.org/gstreamer-1.0/Gst.Pad.add_probe.html) takes 2 arguments - one that explains the [Probe Type](https://valadoc.org/gstreamer-1.0/Gst.PadProbeType.html) and the other the [callback](https://valadoc.org/gstreamer-1.0/Gst.PadProbeCallback.html) function to be called when the probe type is encountered. 
 The probe type is a bit difficult to understand, because very less documentation is available. In my case, I have considered the probe type as `BUFFER`, which will allow to probe the data flow happening on the pads. The other probe types and their usage are as follows: 
 
      INVALID - invalid probe type
      IDLE - probe idle pads and block while the callback is called
      BLOCK - probe and block pads
      BUFFER - probe buffers
      BUFFER_LIST - probe buffer lists
      EVENT_DOWNSTREAM - probe downstream events
      EVENT_UPSTREAM - probe upstream events
      EVENT_FLUSH - probe flush events.
      QUERY_DOWNSTREAM - probe downstream queries
      QUERY_UPSTREAM - probe upstream queries
      PUSH - probe push
      PULL - probe pull
      BLOCKING - probe and block at the next opportunity, at data flow or when idle
      DATA_DOWNSTREAM - probe downstream data (buffers, buffer lists, and events)
      DATA_UPSTREAM - probe upstream data (events)
      DATA_BOTH - probe upstream and downstream data (buffers, buffer lists, and events)
      BLOCK_DOWNSTREAM - probe and block downstream data (buffers, buffer lists, and events)
      BLOCK_UPSTREAM - probe and block upstream data (events)
      EVENT_BOTH - probe upstream and downstream events
      QUERY_BOTH - probe upstream and downstream queries
      ALL_BOTH - probe upstream events and queries and downstream buffers, buffer lists, events and queries
      SCHEDULING - probe push and pull
      
      
 Step 2: Implement the callback function 
  For the current usecase, I have considered the BUFFER probe type to analyse the dataflow passing though the src pads. My application was more focussed towards getting the DTS and PTS of the video frames, for which BUFFER probbe type proved to be viable. Also the return type needs to be chosen carefully,depending on what needs to be done on the data. In my case, I did not need anything more than the timestamps and hence `Gst.PadProbeReturn.OK`, which does nothing but notifies that prpbe callback was successfully handled.
 
     def probe_callback(self,pad,info): 
          dts = info.get_buffer().dts
          pts = info.get_buffer().pts
          return Gst.PadProbeReturn.OK
          
          
 And that's all is required to add probes and analyse the data flowing through the pipeline. The pads can be put on any of the elements in the pipeline and on both source and static pads. The other few probes, which I used included the following usecases : 
 - analyse if the element is idle, i.e. no data flow across the pads
 - PUSH/PULL buffer to get notified in case of any buffer pull and push
 - BLOCK pads, to stop the data flow and insert/delete new elements in the pipeline.
