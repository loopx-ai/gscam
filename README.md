GSCam
===========================================================================================================================

This is a ROS package originally developed by the [Brown Robotics
Lab](http://robotics.cs.brown.edu/) for broadcasting any
[GStreamer](http://gstreamer.freedesktop.org/)-based video stream via the
standard [ROS Camera API](http://ros.org/wiki/camera_drivers). This fork has
several fixes incorporated into it to make it broadcast correct
`sensor_msgs/Image` messages with proper frames and timestamps. It also allows
for more ROS-like configuration and more control over the GStreamer interface.

Note that this pacakge can be built both in a rosbuild and catkin workspaces.



## Features

- [x] Convert YUV to RGB/BGR with low CPU usage (50% on ORIN AGX with 5M Camera, still room for optimization)
- [x] Publish Camera INFO
- [ ] Get the timestamp of the camera itself (gst can get V4L2 timestamp, but need to verify)
- [x] Use hardware acceleration
- [x] Support Nodelet
- [x] Support Resize



How to use
-------------------------

(Tested on ORIN AGX with Ubuntu20.04)

#### Dependencies:

* gstreamer1.0-tools 
* libgstreamer1.0-dev 
* libgstreamer-plugins-base1.0-dev 
* libgstreamer-plugins-good1.0-dev
* v4l-utils

**Ubuntu Install** (≥14.04):

```sh
$ sudo apt-get install gstreamer1.0-tools libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev libgstreamer-plugins-good1.0-dev v4l-utils
```

Just try to simply show the camera:

```shell
$ gst-launch-1.0 v4l2src device={DEVICE} ! 'video/x-raw,format={FORMAT},width={WIDTH},height={HEIGHT},framerate={FPS}/1' ! videoconvert ! fpsdisplaysink video-sink=xvimagesink sync=false
```

**Example**:

```shell
$ gst-launch-1.0 v4l2src device=/dev/video0 ! 'video/x-raw,format=UYVY,width=2880,height=1860,framerate=30/1' ! videoconvert ! fpsdisplaysink video-sink=xvimagesink sync=false
```



#### Usage:

1. Compile the package:

   ```shell
   $ catkin build gscam
   ```

2. Test nvv4l2camerasrc and nvvidconv (**Necessary**):

   ```shell
   $ gst-launch-1.0 nvv4l2camerasrc device={DEVICE} ! 'video/x-raw(memory:NVMM),format={FORMAT},width={WIDTH},height={HEIGHT},framerate={FPS}/1' ! nvvidconv ! video/x-raw,format=BGRx ! videoconvert ! fpsdisplaysink video-sink=xvimagesink sync=false
   ```

   If the image is **distorted**, please execute `$ v4l2-ctl -d /dev/video0 --set-ctrl preferred_stride={STRIDE}`, The `{STRIDE}` parameter is calculated according to `cell({WIDTH} * 2/256) * 256`; For example `{WIDTH}=2880`, then the calculation process is: `cell(2880 * 2/256) * 256 = 23 * 256=5888`. More information please see this [issue](https://forums.developer.nvidia.com/t/500w-camera-problem-failed-to-get-image/195491/17).

 3. See [GMSL sample launch file](launch/gmsl.launch) and run with:

    ```shell
    $ roslaunch gscam gmsl.launch
    ```



#### Notes:

* Currently testing a 5M Camera on ORIN AGX: it occupies about 46% of the CPU single core when it is unloaded, about 70% of the CPU single core when reading image_raw, and about 150% of the CPU when reading compressed.
* Due to the limited output format supported by [nvvidconv](https://docs.nvidia.com/metropolis/deepstream/dev-guide/text/DS_plugin_gst-nvvideoconvert.html), the final conversion to RGB8 and BGR8 still runs on the CPU.



ROS API
----------------

### gscam

This can be run as both a node and a nodelet.

#### Nodes
* `gscam`

#### Topics
* `camera/image_raw`
* `camera/camera_info`

#### Services
* `camera/set_camera_info`

#### Parameters
* `~camera_name`: The name of the camera (corrsponding to the camera info)
* `~camera_info_url`: A url (`file://path/to/file`, `package://pkg_name/path/to/file`) to the [camera calibration file](http://www.ros.org/wiki/camera_calibration_parsers#File_formats).
* `~gscam_config`: The GStreamer [configuration string](http://wiki.oz9aec.net/index.php?title=Gstreamer_cheat_sheet&oldid=1829).
* `~frame_id`: The [TF](http://www.ros.org/wiki/tf) frame ID.
* `~reopen_on_eof`: Re-open the stream if it ends (EOF).
* `~sync_sink`: Synchronize the app sink (sometimes setting this to `false` can resolve problems with sub-par framerates).
* `~image_encoding`: Encoding of the stream. Can be {`rgb8`, `bgr8`, `mono8`}. Defaults to `rgb8`. (Verify with: `$ rostopic echo /camera/image_raw | grep encoding`)

Examples
--------

See example launchfiles and configs in the examples directory. Currently there
are examples for:

* [Video4Linux2](examples/v4l.launch): Standard
  [video4linux](http://en.wikipedia.org/wiki/Video4Linux)-based cameras like
  USB webcams.
    * ***GST-1.0:*** Use the roslaunch argument `GST10:=True` for GStreamer 1.0 variant
* [Nodelet](examples/gscam_nodelet.launch): Run a V4L-based camera in a nodelet
* [Video File](examples/videofile.launch): Any videofile readable by GStreamer
* [DeckLink](examples/decklink.launch):
  [BlackMagic](http://www.blackmagicdesign.com/products/decklink/models)
  DeckLink SDI capture cards (note: this requires the `gst-plugins-bad` plugins)

