How to set up a timeout in Xlib
===============================

The problem
-----------

Xlib event loops hinge around an ``XWindowEvent()`` call or similar, which
returns the next event received from the X server:

```
while (...) {
	XWindowEvent(display, window, &event);
	// process event
}
```

This is enough for all programs that only act in response to something
happening in X Window, such as keystrokes, button presses, window mapping,
unmapping and exposing, selection requests, etc. If nothing like this happens,
``XWindowEvent()`` blocks. The program cannot read from other file descriptors
in the same thread nor do something else after a fixed amount of time.

The solution
------------

The function ``nextevent()`` in ``timeout.c`` shows how to set up a timeout or
read from a file descriptor: it waits for an event from the X server, but also
returns when a timeout expires or data is available from stdin.

```
#define NEXTEVENT_TIMEOUT 0
#define NEXTEVENT_STDIN   1
#define NEXTEVENT_X11     2
int nextevent(Display *dsp, Window win, int timeout, XEvent *evt) {

	...

	while (! XCheckWindowEvent(dsp, win, KeyPressMask, evt)) {

		t = time left to timeout

		select({ConnectionNumber(dsp), STDIN_FILENO}, ..., t);

		if (ConnectionNumber(dsp) can be read)
			continue;

		if (STDIN_FILENO can be read)
			return NEXTEVENT_STDIN;

		return NEXTEVENT_TIMEOUT;
	}
	return NEXTEVENT_X11;
}
```

The ``evt`` parameter is the next event received from the X server only if the
function returns ``NEXTEVENT_X11``. The other possible return values are
``NEXTEVENT_STDIN`` if ``stdin`` is readable and ``NEXTEVENT_TIMEOUT`` if the
timeout expired.

The function selects only ``KeyPressMask`` and only waits for data to be
available from ``stdin`` but easily generalizes to arbitrary event selections
(using a different event mask), to multiple windows (using XPeekEvent() and
XNextEvent()) and multiple file descriptors being ready for reading or writing
(by passing them to ``select()``).

Explanations
------------

Why the loop? Why is ``XCheckWindowEvent()`` called before ``select()`` and not
after? Why calling this non-blocking function instead of its blocking version
``XWindowEvent()`` after ``select()``?

Events are received from the X server through the ``ConnectionNumber(dsp)``
socket, but are not immediately returned to the program. Rather, they are
stored in a queue and returned one by one when the program calls
``XWindowEvent()``, ``XCheckWindowEvent()`` or similar.

As a result, this function may be called when some events are still in the
queue. If ``select()`` is called and no other data comes from the socket, it
blocks. These events are not processed, and the program appears to freeze.

The solution is to check for events before entering the loop. If an event is
available, it is returned immediately and ``select()`` is not called at all.

Another question is why is this a loop in the first place. The answer is that
``select()`` only signals that some data is available on the socket, but does
not tell how much. It may return even a single byte, in theory. In practice, it
may return even if an event is only partially received. It may return a reply
instead of an event. The function should only return when
``XCheckWindowEvent()`` confirms that the event is complete.

The timeout is recalculated every time because of the loop. Since ``select()``
may be called multiple times, the timeout cannot be set the same on each. The
second time, it has be reduced by the elapsed time, or just set to the time
left to the requested ending time.

Higher-level libraries
----------------------

Higher-level libraries allow setting up a callback to be run after a given
time. This is what ``XtAppAddTimeOut()`` does in Xt and ``g_timeout_add()``
does in gtk, for example.
