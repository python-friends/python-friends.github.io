.. title: Looking at microscope slides with computer vision
.. slug: auto-scope-progress
.. date: 2018-10-16 00:00:00 UTC+10:00
.. tags: draft, auto-scope, cv2
.. category: 
.. link: 
.. description: 
.. type: text
.. author: Wytamma

We are making progress on the auto-scope project! The mechanical aspects of the microscope are all finished and working*. We’ve written most of the code to control the x, y, and z of the microscope and we’re starting to move onto the whole slide stitching and machine learning aspects of the project. 

Check out this (oldish) video of the stage moving around. Look at it go!

.. youtube:: 3XsQCkF1SrE
   :align: center
   :width: 100%

.. TEASER_END

Currently, the microscope has stage control (x, y), focus control (z)  and a camera. We’re using an 8MP Raspberry Pi V2 camera to capture images of the slide. Two 28byj-48 stepper motors control the stage (one for x and one for y), and another stepper motor works the focus knob. 

I’ve recently been working on the interactions between the camera and focus to capture tiles and enabling autofocusing. I thought I should detail some of the aspects of this part of the project while they are fresh in my mind.

Because we are using an `open loop system 
<https://en.wikipedia.org/wiki/Motor_control#Open_loop_control>`_ the microscope doesn’t have a way to tell it's current location (only how far it's moved from the starting position). To ensure reproducibility we always initialise the microscope with the focus at maximum height, that way the microscope knows one of its limits, and we can work down from this position.

When taking photos through the microscope with a camera like the RPi V2, the wide lens results in a lot of black surrounding the actual image we want to capture. Ideally, for image stitching, we'd like a square picture with no boarded. We could manually crop this additional border. However, we have Python`
<https://xkcd.com/353/>`_.

.. image:: /images/auto-scope-cam/full.png
    :align: center

So, to automatically extract our square region of interest we first need to calculate a few parameters. To find the width of the circle, we mask the image with a back and white threshold and then move in from the sides until we hit something white. This gives us the diameter, easy. 

.. image:: /images/auto-scope-cam/width.png
    :align: center

To find the center of the circle we use a method from computer vision called `blob centroid detection 
<https://www.learnopencv.com/find-center-of-blob-centroid-using-opencv-cpp-python/>`_. in short, we take the average position of all the points in the circle.

.. image:: /images/auto-scope-cam/center.png
    :align: center

Using the diameter and centre point we can apply a bit of `trigonometry 
<https://en.wikipedia.org/wiki/Special_right_triangle#45%C2%B0%E2%80%9345%C2%B0%E2%80%9390%C2%B0_triangle>`_ to determine the coordinates of a square that is inscribed in the circle. 

.. image:: /images/auto-scope-cam/square.png
    :align: center

We can then feed this square information to the camera, that will zoom in and return nicely cropped images. The above description is implemented in the `calculate_zoom` method of the autoscope `Camera` class.

.. image:: /images/auto-scope-cam/tile.png
    :align: center

Now that we have a tile we need a way to tell if it’s in focus. There are many 'blurriness metrics' for example the `variation of the Laplacian
<https://www.pyimagesearch.com/2015/09/07/blur-detection-with-opencv/>`_. This method returns high values if an image contains high variance in rapid intensity change regions (associated with focused images).

.. image:: /images/auto-scope-cam/focus.png
    :align: center

To find the optimal focal height we need to map the focal plane and try to maximise the variation of the Laplacian (see bar plot). The initial scan of the focal plane is done in large steps down the focal plane until we find a peak. Once the course mapping is complete we then switch to fine steps to map the peak in detail. Now that we have a fine grain map of the focal plane peak we can simply move to focus with the highest variation. This course, fine, find algorithm is implemented in the `auto_focus` method of the autoscope `Autoscope` class.

You can check out all this camera code and more on `GitHub
<https://github.com/python-friends/auto-scope>`_ or `pip install opencv-contrib-python` and try it yourself.

\* My 3D printed gears suffer from a slight backlash that results in small errors when the stepper motors change direction. I think we should move to a timing belt system to prevent this in the future, however, slide scanning is still possible even with these errors. 


