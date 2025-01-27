===============
Getting started
===============

Despiker is a frame-based profiler for the `D programming language <http://dlang.org>`_.
This tutorial will explain how to set up your game/project for profiling with Despiker.

.. raw:: html

   <video style="width:90%;display:block;margin:0 auto" preload="auto" autoplay controls loop poster="../_static/despiker-preview.png">
      <source src="../_static/despiker.webm" type="video/webm">
   </video>

Despiker itself does not record profiling data; it only analyzes and displays it.  The
recording part is handled by `Tharsis.prof <https://github.com/kiith-sa/tharsis.prof>`_,
a profiling library Despiker is based on. To profile your code with Despiker, you first
need to *instrument* it with Tharsis.prof *Zones*. This will be explained further below,
but first, you need to get your game/project to work with Tharsis.prof.

----------------------------------
Using Tharsis.prof in your project
----------------------------------

We assume you are using `dub <http://code.dlang.org/about>`_ (either directly, or through
an `IDE <http://wiki.dlang.org/IDEs>`_ that uses ``dub``) to build your project and manage
its dependencies. If you don't know ``dub``, it is a package manager/build system that has
become the de-facto standard for building D projects.

First, you need to add the ``tharsis-prof`` package to the dependencies of your project.
Add this line to the ``dependencies`` section of your ``dub.json``/``package.json`` file:

.. code-block:: json

   "tharsis-prof": "~>0.4.0"

This means your project will use the newest version of Tharsis.prof in the ``0.4`` series,
but not e.g. ``0.5`` if it is released (incrementing the second digit in ``0.5`` means
breaking changes).


^^^^^^^^^^^^^^^^^^^^^^^
Instrumenting your code
^^^^^^^^^^^^^^^^^^^^^^^

This section covers instrumenting single-threaded game code. There is no tutorial for
profiling multi-threaded code yet, mainly because related API is likely to see large
changes. If you're feeling adventurous, you can look at the `API documentation
<http://defenestrate.eu/docs/tharsis.prof/index.html>`_ of Tharsis.prof. It might be
a good idea to get single-threaded profiling to work first, though.

To record profiling data, you need to create a ``Profiler`` object.  ``Profiler`` records
and strores profiling events such as when a zone is entered or exited (more on zones
below). Add this code somewhere to the initialization code of your game, or wherever you
want profiling to start (``Profiler`` needs memory, so you might not want to have it
running by default; it may make sense to only construct a Profiler when needed, e.g.
based on user input).

.. code-block:: d

    // 64 MB. Profiler.maxEventBytes is the minimum size of memory that can be passed 
    // to Profiler constructor. It's very small (below 1kB), but it's good practice to 
    // explicitly use it to ensure we always have enough memory.
    // 
    // Depending on the number of zones per frame, 64MB may be overkill or too little; you 
    // might need to experiment to find out how much you really need.
    enum storageLength = Profiler.maxEventBytes + 64 * 1024 * 1024;
    // If you want manual allocation, you can do this (but make sure to free() it later!)
    // ubyte[] storage  = (cast(ubyte*)malloc(storageLength))[0 .. storageLength];
    ubyte[] storage  = new ubyte[storageLength];

    // If you want manual allocation, you can do this:
    // import std.typecons;
    // auto profiler = scoped!Profiler(storage);
    auto profiler = new Profiler(storage);


.. admonition:: Profiler memory usage

   ``Profiler`` never allocates memory; it only uses memory passed by the constructor.  It
   is up to the caller *how* to allocate the memory. ``Profiler`` will eventually run out 
   of space and quietly stop recording; this can be detected by ``Profiler.outOfSpace()``. 
   ``Profiler.reset()`` can then be used to clear data and start recording anew.
   Unfortulately, Despiker **does not support** ``Profiler`` resets *yet*, so the duration 
   of time you can profile is limited by the memory you provide to ``Profiler``.


Now that you have a ``Profiler``, you can instrument your code. Tharsis.prof and Despiker 
work by keeping track of precise times when *zones* in core were entered and exited. 
This work is done by the ``Zone`` struct. The first ``Zone`` we need is one that wraps 
all (or almost all, depending on your needs) execution in a frame. Assuming your main
game loop looks something like this:

.. code-block:: d

   for(;;)
   {
        // .. frame code here (game logic, rendering, etc.)
   }

You can add a ``"frame"`` zone like this:  

.. code-block:: d

   for(;;)
   {
        // Passing profiler we constructed above
        Zone frameZone = Zone(profiler, "frame");
        // .. frame code here (game logic, rendering, etc.)
   }

``Zone`` records time when it is constructed/*entered*, and when it is destroyed/*exited*
(when exiting the scope by default).  Note the name, or *zone info string*: By default,
Despiker recognizes zones with info string ``"frame"`` to represent frames, and no other
zones should have this info string.

.. admonition:: Optional profiling

   As already mentioned above, you probably don't want to run Profiler by default as it
   needs considerable amount of memory.  ``Zone`` will ignore and do nothing if the
   profiler reference passed to it is ``null``. So if you want profiling to be optional,
   you can keep your ``Zone`` instances in your code and simply set the ``profiler`` to
   null when you're not profiling.

To profile any other parts of code you are interested in, just add ``Zone`` instances to
their enclosing scopes. Interesting examples may be draw calls, collision detection,
updates of your game's entities and so on, depending on your game. As you view your frames
with Despiker, you will notice any gaps where you might want to add more ``Zone``
instances.


-------------------
Setting up Despiker
-------------------

Once you have a few ``Zones`` in your code, you need to get Despiker to view them.

Despiker requires OpenGL 3.3 for drawing at the moment. This requirement may change in
future. It also requires the `SDL 2 <http://libsdl.org>`_ library.

On Linux, you will need to install SDL 2 to run Despiker. For example on
Debian/Mint/Ubuntu::

   sudo apt-get install libsdl2-dev

On Windows you will need a SDL2 DLL file (once there are official builds for Windows, 
this will be included).

If there is a binary release for your system, you can download it directly. Otherwise you 
will need to build Despiker from scratch.

^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Downloading a binary release
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

For now, there are only binaries for x86-64 Linux, which is the only platform Despiker has
been tested on. On other systems you will need to build Despiker from scratch (for now).

You can get the binary archive for the newest release `here
<https://github.com/kiith-sa/tharsis.prof/releases/latest>`_.


^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Building Despiker from scratch
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Despiker uses `dub <http://code.dlang.org/about>`_ as the build system and requires DMD
2.066 (or equivalent LDC or GDC) for compilation. (`D compiler downloads
<http://dlang.org/download.html>`_)

Source code for the newest Despiker release can be downloaded `here
<https://github.com/kiith-sa/tharsis.prof/releases/latest>`_.

Once you've installed ``dub`` and a D compiler and downloaded Despiker source archive,
extract the source code and open the extracted directory in console. To build despiker, 
type::

   dub build

Before building, ``dub`` will automatically download any packages Despiker depends on
(this means you do need an Internet connection to build Despiker).

When the above command finishes, you should have a binary file called ``despiker`` or 
``despiker.exe`` in your directory.


^^^^^^^^^^^^^^^^^^^^^^^^^
Placing Despiker binaries
^^^^^^^^^^^^^^^^^^^^^^^^^

Despiker is (currently) launched from the profiled game by Tharsis.prof. Tharsis.prof
looks for Despiker in following directories:

* Directory specified explicitly in code, if any
* Working directory (the directory the game was launched from)
* Directory with the game binary 
* ``PATH``

You can "install" Despiker by copying it to any of these directories.  If you've
downloaded a binary archive, extract its contents to the game directory; if you've built
it from source, copy the ``despiker``/``despiker.exe`` and ``DroidSans.ttf`` files.



---------------------------------------------
Launching Despiker and sending profiling data
---------------------------------------------

Tharsis.prof can launch and send data to Despiker using the ``DespikerSender`` class.
``DespikerSender`` can be initialized after the ``Profiler``:

.. code-block:: d

   auto sender = new DespikerSender([profiler]);

``DespikerSender`` constructor takes an array of ``Profiler`` references. Profiling data
recorded by all of these ``Profiler`` instances will be sent to Despiker, which is useful
e.g. for profiling multithreaded code (with one ``Profiler`` per thread). However, it
should be noted that ``Despiker`` assumes that frames from these profilers are aligned in
time; if you use multiple ``Profilers``, a frame zone in the first profiler should have
a corresponding frame zone in all other profilers.  This limitation may be replaced by
something smarter in future. Of course, if you only need to profile a single thread, this
shouldn't be an issue for you.

.. admonition:: Implementation notes

   Currently, DespikerSender launches Despiker and pipes profiling data through its
   standard input. This is the main reason why the only practical (cross-platform) way to
   launch Despiker is from the profiled game. In future, a socked-based implementation may
   be added, which could make it possible to launch Despiker stand-alone.


Next you need to call ``DespikerSender.update()`` whenever you might want
``DespikerSender`` to send data to Despiker. Once Despiker is running, ``update()`` 
sends new profiling data to it and checks if it has been closed. To get smooth, real-time
profiling, it's good to call ``update()`` once per frame. Note that ``update()`` must 
not be called if *any* of the profilers passed to ``DespikerSender`` constructor are in
a zone. In our case, we need to exit the ``"frame"`` zone by destroying it before updating
the ``DespikerSender``:

.. code-block:: d

   destroy(frameZone);
   sender.update();


To launch Despiker, use ``DespikerSender.startDespiker()``. Note that this should't be
called if ``DespikerSender`` is already ``sending()`` to a previously launched Despiker.
You can 'forget' and stop sending to a previously launched Despiker by
``DespikerSender.reset()``.

.. code-block:: d

   // If you want to explicitly specify Despiker path:
   // sender.startDespiker("path/to/despiker.exe"); 
   sender.startDespiker(); 

``DespikerSender`` will first send all profiling data recorded so far; this may result in 
a slight hang when Despiker starts. After that, new profiling data will be sent gradually,
with each ``DespikerSender.update()`` call, and Despiker will show a graph for the current
frame in real-time (this may seem like a flickering blur if the FPS is high enough).

To view frames and their zones, you can use the GUI or these controls:

========================= ===========================================
Control                   Action
========================= ===========================================
``Space``                 Pause/resume current (real-time) frame view
``H``/``L``               Previous/next frame
``W``/``D``, ``RMB`` drag Panning
``-``/``+``, mouse wheel  Zooming
``1``                     Jump to the worst/slowest frame
========================= ===========================================
