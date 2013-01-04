.. -*- fill-column: 120; -*-

Near-term actions
=================

* COPY THE ti-dvsdk-dm368-evm ONTO A FEW DVDs AND PUT THEM IN SAFE PLACES. Include this git repo.
* Get the toolchain back up and running. Use `Code Sourcery`_ which is `documented on this wiki`_.
* Test the toolchain by building the DVSDK. Instructions for this appear in the two PDFs that I put on my Google Drive.
* Figure out how the Buffer struct refers to actual image bytes.
* Figure out what is required to access physical memory. Look at the use of ``mmap`` in `my work on the Raspberry Pi`_.
  Something similar is happening in `Framecopy_accel.c`_.

.. _`my work on the Raspberry Pi`: https://github.com/wware/rpi-hacking/blob/master/dev-mem/tryit.c
.. _`Framecopy_accel.c`: https://github.com/wware/ti-dvsdk-dm368-evm/blob/master/dmai_2_20_00_15/packages/ti/sdo/dmai/linux/dm365/Framecopy_accel.c
.. _`TI has thoughts`: http://processors.wiki.ti.com/index.php/Linux_Toolchain
.. _`Code Sourcery`: http://tw.myblog.yahoo.com/stevegigijoe/article?mid=366
.. _`documented on this wiki`: http://www.nas-central.org/wiki/Setting_up_the_codesourcery_toolchain_for_X86_to_ARM9_cross_compiling

Studying the DM368 EVM code from TI
===================================

I want to do some interaction with video data on the DM368 EVM board. The software in TI's SDK makes it a little
non-obvious how to go about this because they have layers and layers of indirection, and a pointer to an actual
buffer is buried away in some obscure C header file among thousands, as if accessing video data is something that
nobody has ever wanted to do. Go figure.

Some relevant stuff online:

* http://processors.wiki.ti.com/index.php/CMEM_Overview
* http://processors.wiki.ti.com/index.php/DSPLink_POOL_Module_Overview
* http://processors.wiki.ti.com/index.php/EricScottVideos - Scott Specker and Eric Wilbur provide a 30-minute overview
  of the Codec Engine used with TI's DaVinci and OMAP processors.
* http://processors.wiki.ti.com/index.php/Linux_Toolchain
* http://processors.wiki.ti.com/index.php/DaVinci_GIT_Linux_Kernel

Getting started
===============

I did the following things to set up a git repo in the ``~/ti-dvsdk-dm368-evm`` directory.

::

 git init
 touch README.rst     # empty
 git add README.rst
 git commit
 # Add a bunch of notes to README.rst
 git add makehtml.sh
 git add $(find dmai_2_20_00_15/ codec-engine_2_26_02_11/ \
     example-applications/ dvsdk-demos_4_02_00_01/ -name '*.[ch]')
 git commit -a

The area of the code most relevant to my applicaion is in ``dvsdk-demos_4_02_00_01/dm365/encodedecode/`` directory so
I'll try to connect everything to that code. In preliminary investigation, I've discovered two interesting functions
that are defined in multiple places::

 UInt32 Memory_getBufferPhysicalAddress(Ptr virtualAddress, Int sizeInBytes, Bool *isContiguous);
 Ptr Memory_getBufferVirtualAddress(UInt32 physicalAddress, Int sizeInBytes);

They are defined differently for different platforms (Linux, WinCE, BIOS, etc.):

* ``codec-engine_2_26_02_11/packages/ti/sdo/ce/osal/wince/Memory_cmem.c``
* ``codec-engine_2_26_02_11/packages/ti/sdo/ce/osal/linux/Memory_noOS.c``
* ``codec-engine_2_26_02_11/packages/ti/sdo/ce/osal/linux/Memory_cmem.c``
* ``codec-engine_2_26_02_11/packages/ti/sdo/ce/osal/bios/Memory_BIOS.c``
* ``codec-engine_2_26_02_11/packages/ti/sdo/ce/osal/Memory.h``
* ``codec-engine_2_26_02_11/packages/ti/sdo/ce/osal/noOS/Memory_noOS.c``

And then in ``dmai_2_20_00_15/packages/ti/sdo/dmai/Buffer.c``, I found more interesting functions that do different
things with Buffers in memory::

 Bool Buffer_isReference(Buffer_Handle hBuf);
 BufTab_Handle Buffer_getBufTab(Buffer_Handle hBuf);
 Buffer_Handle Buffer_clone(Buffer_Handle hBuf);
 Buffer_Handle Buffer_create(Int32 size, Buffer_Attrs *attrs);
 Buffer_Type Buffer_getType(Buffer_Handle hBuf);
 Int Buffer_copy(Buffer_Handle hSrcBuf, Buffer_Handle hDstBuf);
 Int Buffer_delete(Buffer_Handle hBuf);
 Int Buffer_getId(Buffer_Handle hBuf);
 Int Buffer_setSize(Buffer_Handle hBuf, Int32 size);
 Int Buffer_setUserPtr(Buffer_Handle hBuf, Int8 *ptr);
 Int Buffer_setVirtualSize(Buffer_Handle hBuf, Int32 size);
 Int32 Buffer_getNumBytesUsed(Buffer_Handle hBuf);
 Int32 Buffer_getPhysicalPtr(Buffer_Handle hBuf);
 Int32 Buffer_getSize(Buffer_Handle hBuf);
 Int32 _Buffer_getOriginalSize(Buffer_Handle hBuf);
 Int8 *Buffer_getUserPtr(Buffer_Handle hBuf);
 UInt16 Buffer_getUseMask(Buffer_Handle hBuf);
 Void Buffer_freeUseMask(Buffer_Handle hBuf, UInt16 useMask);
 Void Buffer_getAttrs(Buffer_Handle hBuf, Buffer_Attrs * attrs);
 Void Buffer_print(Buffer_Handle hBuf);
 Void Buffer_resetUseMask(Buffer_Handle hBuf);
 Void Buffer_setNumBytesUsed(Buffer_Handle hBuf, Int32 numBytes);
 Void Buffer_setUseMask(Buffer_Handle hBuf, UInt16 useMask);
 Void _Buffer_setBufTab(Buffer_Handle hBuf, BufTab_Handle hBufTab);
 Void _Buffer_setId(Buffer_Handle hBuf, Int id);

Connecting back to the area of interest
=======================================

My goal is to find out how the C code accesses image data in memory, so this was a start. In
``dvsdk-demos_4_02_00_01/dm365/encodedecode/video.c``, I find this, which looks interesting::

 /* Allocate buffer for encoded data * /
 hEncBuf = Buffer_create(Vdec2_getInBufSize(hVd2), &bAttrs);

and that's used in the same file here::
 
 /* Encode the video buffer * /
 if (Venc1_process(hVe1, hVidBuf, hEncBuf) < 0) {
     ERR("Failed to encode video buffer\n");
     return FAILURE;
 }

``Venc1_process`` is defined on line 97 of ``dmai_2_20_00_15/packages/ti/sdo/dmai/ce/Venc1.c``. It passes the buck to
``VIDENC1_process`` defined on line 217 of ``codec-engine_2_26_02_11/packages/ti/sdo/ce/video1/videnc1.c``.

The next interesting thing is this::

 IVIDENC1_Fxns *fxns =
     (IVIDENC1_Fxns * )VISA_getAlgFxns((VISA_Handle)handle);
 IVIDENC1_Handle alg = VISA_getAlgHandle((VISA_Handle)handle);
 ....
     VISA_enter((VISA_Handle)handle);
     retVal = fxns->control(alg, id, dynParams, status);
     VISA_exit((VISA_Handle)handle);

which sends us off to ``xdais_6_26_01_03/packages/ti/xdais/dm/ividenc1.h`` and
``codec-engine_2_26_02_11/packages/ti/sdo/ce/visa.c`` to observe that we are invoking the DSP from the ARM CPU. That's
nice but it's a tangent, so back to ``dvsdk-demos_4_02_00_01/dm365/encodedecode/``.

I want access to the data immediately after video capture. This happens in ``capture.c`` when it calls ``Capture_get``
defined at ``dmai_2_20_00_15/packages/ti/sdo/dmai/linux/dm365/Capture.c`` line 746::

 Int Capture_get(Capture_Handle hCapture, Buffer_Handle *hBufPtr);

The captured video frame is stored in ``hCapBuf`` in the ``captureThrFxn`` thread function, and at that same point we
also have the width, height, and buffer size.

So what to do next
==================

I think it makes sense to capture the frame as normal, then copy it into another buffer, and allow the original buffer
to go through the normal signal processing chain. My algorithm collects information from the copied buffer, and I'll
need to dump it somewhere it can be viewed. Eventually, I need to put the whole application together.

I need to know what's inside the ``Buffer`` data structure and how I can read bytes out of it and write bytes into it.
Here are two files of interest, with interesting definitions in them:

* dmai_2_20_00_15/packages/ti/sdo/dmai/Buffer.h

  - typedef struct Buffer_Attrs { ... };
  - typedef struct _Buffer_Object \*Buffer_Handle;

* dmai_2_20_00_15/packages/ti/sdo/dmai/priv/_Buffer.h

  - typedef struct _Buffer_State { ... };
  - typedef struct _Buffer_Object { ... };
  - typedef struct _BufferGfx_Object { ... };

So let's look more closely at the most likely suspect::

 typedef struct _Buffer_Object {
     Buffer_Type             type;
     _Buffer_State           origState;
     _Buffer_State           usedState;
     Memory_AllocParams      memParams;
     Int8                   *userPtr;
     Int32                   physPtr;
     Int                     id;
     Bool                    reference;
     BufTab_Handle           hBufTab;
     Int32                   virtualBufferSize;
 } _Buffer_Object;




Buffer.h File Reference
=======================

::

 #include <xdc/std.h>
 #include <ti/sdo/ce/osal/Memory.h>
 #include <ti/sdo/dmai/Dmai.h>
 #include <ti/sdo/dmai/BufTab.h>

Data Structures
---------------

* ``struct Buffer_Attrs`` -- Attributes used when creating a Buffer instance.


Typedefs
--------

* ``typedef struct _Buffer_Object * Buffer_Handle`` -- Handle through which to reference a Buffer instance.

Enumerations
------------

::

 enum  Buffer_Type_ {
   Buffer_Type_BASIC_ = 0,
   Buffer_Type_GRAPHICS_ = 1,
   Buffer_Type_COUNT_
 }

Types of Buffers.

Functions
---------

* ``Buffer_Handle Buffer_create (Int32 size, Buffer_Attrs *attrs)`` -- Creates and allocates a contiguous Buffer.
* ``Buffer_Handle Buffer_clone (Buffer_Handle hBuf)`` -- Creates and clone of an existing Buffer. Only the attributes used
  while creating the cloned Buffer will be used.
* ``Void Buffer_print (Buffer_Handle hBuf)`` -- Prints information about a buffer.
* ``Int Buffer_delete (Buffer_Handle hBuf)`` -- Deletes and frees a contiguous Buffer.
* ``Void Buffer_getAttrs (Buffer_Handle hBuf, Buffer_Attrs *attrs)`` -- Get the Buffer_Attrs corresponding to existing buffer.
* ``Void Buffer_setUseMask (Buffer_Handle hBuf, UInt16 useMask)`` -- Set the current use mask.
* ``Void Buffer_freeUseMask (Buffer_Handle hBuf, UInt16 useMask)`` -- Free bits in the current use mask. When the resulting use mask is 0, the
  Buffer is considered free.
* ``Void Buffer_resetUseMask (Buffer_Handle hBuf)`` -- Set the current use mask to the original use mask, essentially marking the
  Buffer as busy.
* ``UInt16 Buffer_getUseMask (Buffer_Handle hBuf)`` -- Get the current use mask of a Buffer.
* ``Int Buffer_getId (Buffer_Handle hBuf)`` -- Get the id of a Buffer. The id identifies a Buffer in a BufTab.
* ``Int8 * Buffer_getUserPtr (Buffer_Handle hBuf)`` -- Get the user pointer of the Buffer. This pointer can be used to access the
  Buffer using the CPU.
* ``Int32 Buffer_getPhysicalPtr (Buffer_Handle hBuf)`` -- Get the physical pointer of the Buffer. This pointer can be used by device
  drivers and DMA to access the Buffer.
* ``Int32 Buffer_getSize (Buffer_Handle hBuf)`` -- Get the size of a Buffer.
* ``Buffer_Type Buffer_getType (Buffer_Handle hBuf)`` -- Get the type of a Buffer.
* ``Int32 Buffer_getNumBytesUsed (Buffer_Handle hBuf)`` -- When a DMAI module has processed data and written it to a Buffer, it
  records the actual number of bytes used (which may or may not be the same as
  the size).
* ``Void Buffer_setNumBytesUsed (Buffer_Handle hBuf, Int32 numBytes)`` -- Set the number of bytes used in a Buffer. If you process data outside of
  DMAI, call this function to tell the DMAI modules how many bytes it should
  process in the Buffer.
* ``Int Buffer_setUserPtr (Buffer_Handle hBuf, Int8 *ptr)`` -- Set the User pointer for a Buffer reference.
* ``Int Buffer_setSize (Buffer_Handle hBuf, Int32 size)`` -- Set the size of a Buffer reference.
* ``Int Buffer_setVirtualSize (Buffer_Handle hBuf, Int32 size)`` -- Set the virtual size of a Buffer.
* ``Bool Buffer_isReference (Buffer_Handle hBuf)`` -- Investigate whether a Buffer instance is a reference or not.
* ``BufTab_Handle Buffer_getBufTab (Buffer_Handle hBuf)`` -- Get the BufTab instance which a Buffer belongs to, if any.
* ``Int Buffer_copy (Buffer_Handle hSrcBuf, Buffer_Handle hDstBuf)`` -- Copies Buffer object from source to destination.

Variables
---------

* ``const Memory_AllocParams Buffer_Memory_Params_DEFAULT`` -- The default parameters for the Memory module while creating a Buffer.
* ``const Buffer_Attrs Buffer_Attrs_DEFAULT`` -- The default parameters when creating a Buffer.

