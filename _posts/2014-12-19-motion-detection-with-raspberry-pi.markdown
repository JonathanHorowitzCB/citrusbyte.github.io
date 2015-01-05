---
layout: post
title:  "Motion detection with Raspberry PI"
description: "Explore motion detection using two methods: manual calculations and H.264 motion vectors."
date:   2014-12-19 12:52:00
categories: code 
redirect_from: "/code/2014/12/19/pimotion/"
author: jhenkens
---

Motion detection is a tricky problem to solve, the key is to balance sensitivity so that it doesn't catch light changes but still catches movements in the frame. In this article I will describe 2 methods to achieve motion detection: calculating frame differences manually and harnessing the power of the H.264 encoder. Finally, we'll store the results online using using a connected devices (Internet-of-Things) platform from AT&T.

Full source code is available at the end of the article.

# The setup

[Raspberry PI](http://www.raspberrypi.org/) is a low cost single-board computer that is built with the intention of teaching computer science in schools. With the [camera module](http://www.raspberrypi.org/products/camera-module/) and the small footprint of the Raspberry PI this setup is the perfect platform for our motion detection system.

Don't have a Raspberry Pi or the camera module? You can get them here:

* [Raspberry Pi Model B+ (B PLUS) 512MB Computer Board](http://www.amazon.com/Raspberry-Pi-Model-512MB-Computer/dp/B00LPESRUK/ref=sr_1_2?s=pc&ie=UTF8&qid=1419632605&sr=1-2&keywords=raspberry+pi) on Amazon
* [Raspberry PI 5MP Camera Board Module](http://www.amazon.com/Raspberry-5MP-Camera-Board-Module/dp/B00E1GGE40/ref=sr_1_1?s=pc&ie=UTF8&qid=1419632621&sr=1-1&keywords=raspberry+pi+camera+module) on Amazon

# Plan of action

The goal of the project is to monitor the camera for movement. Once we detect movement in the video stream we will switch over to still camera mode and take photos over the span of several seconds to capture the movement in high resolution snapshots.

We will be using Python as the programming language. Python is a clean language, has a large collection of community developed libraries, and has great support.

> Fun fact: Eben Upton and his team envisioned Python as the introduction to programming Raspberry Pi's which is where the "Pi" in the name comes from.

# Method 1: Manual snapshots

Manual motion calculation requires us to take continuous photos. We use 2 buffers, one contains the current raw data and the other one contains the previous frame. After capturing each frame we compare the 2 buffers against each other. For performance reasons the snapshots taken every second are small thumbnail sized photos.

We loop through all the pixels in the green channel of the 2 buffers because the green channel is the highest quality channel. Each pixel value from frame 1 is compared to the matching pixel in frame 2. If the value differs more than the threshold the pixel is considered different and added to the count. Once this process is completed and the total number of changed pixels is higher than the sensitivity we consider movement to be detected and take a full snapshot.

{% highlight Python linenos %}
threshold = 10
sensitivity = 20

def captureFrame():
    command = "raspistill -w %s -h %s -t 0 -e bmp -o -" % (100, 75)
    imageData = StringIO.StringIO()
    imageData.write(subprocess.check_output(command, shell=True))
    imageData.seek(0)
    im = Image.open(imageData)
    buffer = im.load()
    imageData.close()
    return im, buffer

## Capture the first frame
image1, buffer1 = captureFrame()

while (True):
    # Capture a second frame
    image2, buffer2 = captureFrame()

    # Calculate difference
    changedPixels = 0
    for x in xrange(0, 100):
        for y in xrange(0, 75):
            # Just check green channel as it's the highest quality channel
            diff = abs(buffer1[x,y][1] - buffer2[x,y][1])
            if diff > threshold:
                changedPixels += 1

    if changedPixels > sensitivity:
        lastCapture = time.time()
        # Capture full picture if threshold is reached
        saveImage(1280, 960)

    # Swap the buffers for the next frame comparison
    image1 = image2
    buffer1 = buffer2
{% endhighlight %}

## Manual Snapshot Final Thoughts

The idea is simple and straightforward but there is one big disadvantage to this approach. Calculating movement by comparing raw pixel values makes it overly sensitive to static, brightness changes and minor movements. These are not the movements that we like to detect which results in a lot of false positives.

# Method 2: Using motion vectors

The first thing I wanted to do is to prevent using shell commands, after researching online I found the [picamera python library](http://picamera.readthedocs.org/en/release-1.8/) that achieves this requirement.

The Raspberry PI's camera is capable of outputting the motion vector estimates that the H.264 encoder calculates while compressing video. We will harness this data by outputting it to a separate video channel stream.

{% highlight Python linenos %}
class PiMotion:
    def __init__(self, verbose=False, post_capture_callback=None):
        self.verbose = verbose
        self.post_capture_callback = post_capture_callback

    def start(self):
        with picamera.PiCamera() as camera:
            camera.resolution = (1280, 960)
            camera.framerate = 10

            handler = CaptureHandler(camera, self.post_capture_callback)

            self.__print('Waiting 2 seconds for the camera to warm up')
            time.sleep(2)

            try:
                self.__print('Started recording')
                # We can dump the actual video data since we don't use it
                camera.start_recording(
                    '/dev/null', format='h264',
                    motion_output=MyMotionDetector(camera, handler)
                )

                while True:
                    handler.tick()
                    time.sleep(1)
            finally:
                camera.stop_recording()
                self.__print('Stopped recording')
{% endhighlight %}

To hook into the frame by frame data from the picamera library we have to attach an output handler to the picamera object. We created our own to analyze the vector data generated by the H.264 encoder of the Raspberry PI camera.

{% highlight Python linenos %}
class MyMotionDetector(picamera.array.PiMotionAnalysis):
    def __init__(self, camera, handler):
        super(MyMotionDetector, self).__init__(camera)
        self.handler = handler
        self.first = True

    # This method is called after each frame is ready for processing.
    def analyse(self, a):
        a = np.sqrt(
            np.square(a['x'].astype(np.float)) +
            np.square(a['y'].astype(np.float))
        ).clip(0, 255).astype(np.uint8)
        # If there are 50 vectors detected with a magnitude of 60.
        # We consider movement to be detected.
        if (a > 60).sum() > 50:
            # Ignore the first detection, the camera sometimes
            # triggers a false positive due to camera warmup.
            if self.first:
                self.first = False
                return
            # The handler is explained later in this article
            self.handler.motion_detected()
{% endhighlight %}

The output handler requires that the code in the analyze callback to be executed before the next frame is getting rendered. If that doesn't happen you get dropped frames and unreliable behavior. because of this, we can't put our snapshot and upload logic in the analyse method since this takes several seconds to be completed.

Our solution is simple and elegant, in our main loop, is to make a second handler check for a '*motion has detected*' flag. Our original motion handler keeps handling the events from the encoder but just sets the flag in our second handler. This is quick and does not block the output handler.

Once our second handler detects that the flag has been set to true, the capture logic initiates and ignores any addition movement until the capture process has been completed.

{% highlight Python linenos %}
class CaptureHandler:
    def __init__(self, camera, post_capture_callback=None):
        self.camera = camera
        self.callback = post_capture_callback
        self.detected = False
        self.working = False
        self.i = 0

    def motion_detected(self):
        if not self.working:
            self.detected = True

    def tick(self):
        if self.detected:
            print "Started working on capturing"
            self.working = True
            self.detected = False
            self.i += 1

            path = "captures/%s/" % datetime.datetime.now().isoformat()

            os.mkdir(path)

            # Show the preview window on a connected display device
            self.camera.start_preview()

            # Take 16 photos to capture the movement
            for x in range(1, 16):
                filename = "detected-%02d.jpg" % x
                # Because we are continuously capturing video data
                # we have to use the video port to capture our photos
                self.camera.capture(path + filename, use_video_port=True)
                print "Captured " + filename

            self.camera.stop_preview()

            print "Generating the montage"
            montage_file = path + 'montage.jpg'
            # We use imagemagick's montage to create a montage of the 16 photos
            subprocess.call("montage -border 0 -background none -geometry 240x180 " + path + "* " + montage_file, shell=True)

            print "Finished capturing"

            if self.callback:
                self.callback(montage_file)

            self.working = False
{% endhighlight %}


## Motion Vector Method Final Thoughts

Using the motion channel was a good decision, it removes the disadvantages of brightness changes and noise in the pixel data by harnessing the power of the H.264 encoder. By using the picamera library we also removed the use of shell commands.

# Uploading to AT&T's M2X

We now have a montage file that has a grid of all the snapshots taken when motion was detected. Next we will wrap the project up by uploading the final image to our [AT&T's M2X](https://m2x.att.com) feed.

First we have to upload the image to a public storage service, for this project I decided to use [CloudApp](https://www.getcloudapp.com/).

With the existence of the post_capture_callback hook in our Pimotion class, implementing the upload logic is very simple. In our custom callback method we use the [CloudApp API](https://github.com/cloudapp/api/blob/master/README.md#the-cloudapp-api) library (included in the project files) to upload our montage file to their service.

{% highlight Python linenos %}
from pimotion import PiMotion
from cloudapp import CloudAppAPI
from cloudapp.exceptions import CloudAppHttpError
from requests.exceptions import HTTPError, RequestException
import ConfigParser

Config = ConfigParser.ConfigParser()
Config.read('settings.cfg')

def callback(path):
    try:
    	# Upload montage file to our CloudApp account
        api = CloudAppAPI(Config.get('cloudapp', 'username'), Config.get('cloudapp', 'password'))
        url = api.upload(path)
        print 'Public URL: ' + url
    except HTTPError, e:
        print 'ERROR: ' + e.message
    except RequestException, e:
        print 'ERROR: ' + str(e)
    except CloudAppHttpError, e:
        print 'CloudApp ERROR:' + e.message

motion = PiMotion(verbose=True, post_capture_callback=callback)
motion.start()
{% endhighlight %}

The source code of the CloudApp API class can be found on [GitHub](https://github.com/citrusbyte/pimotion/blob/master/pimotion/cloudapp/__init__.py).

With the public URL of the montage file available to us, we can complete the last step of our upload process. POSTing the URL to our [M2X](https://m2x.att.com) device. AT&T provides an official [M2X Python library](https://m2x.att.com/developer/client-libraries#python) that we will be using in this project.

{% highlight Python linenos %}
from pimotion import PiMotion
from cloudapp import CloudAppAPI
from cloudapp.exceptions import CloudAppHttpError
from m2x.client import M2XClient
from m2x.errors import APIError
from requests.exceptions import HTTPError, RequestException
import ConfigParser

Config = ConfigParser.ConfigParser()
Config.read('settings.cfg')

def callback(path):
    try:
        api = CloudAppAPI(Config.get('cloudapp', 'username'), Config.get('cloudapp', 'password'))
        url = api.upload(path)
        print 'Public URL: ' + url

        # POST public URL to our M2X Feed
        client = M2XClient(Config.get('m2x', 'api_key'))
        feed = client.feeds.details(Config.get('m2x', 'feed_id'))
        result = feed.add_values({Config.get('m2x', 'stream'): [{'value': url}]})
        print "Posted URL to M2X channel %s" % Config.get('m2x', 'stream')
    except HTTPError, e:
        print 'ERROR: ' + e.message
    except RequestException, e:
        print 'ERROR: ' + str(e)
    except APIError, e:
        errors = ', '.join("%s: %r" % (key, val) for (key, val) in e.errors.iteritems())
        print "ERROR: %s - %s" % (e.message, errors)
    except CloudAppHttpError, e:
        print 'CloudApp ERROR:' + e.message

motion = PiMotion(verbose=True, post_capture_callback=callback)
motion.start()

{% endhighlight %}


# Final Thoughts

Using the Raspberry PI as the platform is a perfect fit for a motion detection system. The small footprint allows us to embed the Pi in a small encasing to monitor movement without bulky and expensive hardware.

Using the motion vector data channel from the H.264 encoder allowed us to detect motion with a fairly accurate result. The python lib picamera made using the various Pi camera module functions a breeze.

With AT&T's M2X python library, uploading our final montage photo to our device was possible with just three (3) lines of code. 

The full source code with installation instructions can be found here: [https://github.com/citrusbyte/pimotion](https://github.com/citrusbyte/pimotion)
