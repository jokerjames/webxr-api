# WebXR draft (do not implement)

This document is based off of the [Editor's draft of WebVR](https://w3c.github.io/webvr/spec/latest/), modified to support both VR and AR.

The major concepts are:

*XRDisplay*: a particular device and method for rendering XR layers (e.g. Daydream, Vive, Rift, Hololens, GearVR, Cardboard, or magic window)

*Reality*: the rearmost information shown in a display (e.g. the real world in a passthrough display, a virtual reality in a HMD, a camera view in a magic window)

*XRSession*: an interface to a display for rendering onto layers and requesting changes to the current Reality

*XRLayer*: Each XRSession has a XRLayer that exposes the particular context (e.g. a WebGL context) for rendering.

*XRPresentationFrame*: Information needed to render a single graphics frame into a layer, including pose information, as well as sensor data like point clouds and anchors.

The typical application will request an XRSession from an XRDisplay, request a change to the Reality if necessary, then repeatedly request a XRPresentationFrame with which to render into the XRSession's XRLayer.

Applications that require a dedicated virtual reality can request one from the XRSession and then replace the session's default Reality.

The UA will be in control of which Reality is active, the render order of the session layers, and which session will receive the user's input events.

_"VR" in names have been changed to "XR" to indicate that they are used for both VR and AR._

## XR

	interface XR {
		Promise<sequence<XRDisplay>> getDisplays();

		attribute EventHandler ondisplayconnect;
		attribute EventHandler ondisplaydisconnect;
	};

## XRDisplay

	interface XRDisplay : EventTarget {
		readonly attribute DOMString displayName;
		readonly attribute boolean isExternal;

		Promise<boolean> supportsSession(XRSessionCreateParametersInit parameters);
		Promise<XRSession> requestSession(XRSessionCreateParametersInit parameters);

		attribute EventHandler ondeactivate;
	};

Each XRDisplay represents a method of using a specific type of hardware to render AR or VR realities and layers.

_The VRDevice interface was renamed XRDisplay to denote that it is specifically for graphical display types and not other types of devices._

A Pixel XL could expose several displays: a flat display, a magic window display, a Cardboard display, and a Daydream display.

A PC with an attached HMD could expose a flat display and the HMD.

A PC with no attached HMD could expose single a flat display.

A Hololens could expose a single passthrough display.

- How do we support calibration?
- How do we support orientation reset?
- What should we do about area description files?

## XRSession

	interface XRSession : EventTarget {
		readonly attribute XRDisplay display;
		readonly attribute XRSessionCreateParameters createParameters;

		readonly attribute <sequence <Reality>> realities; // All realities available to this session
		readonly attribute Reality reality; // Defaults to most recently used Reality

		attribute XRLayer baseLayer;
		attribute double depthNear;
		attribute double depthFar;


		Reality createVirtualReality(DOMString name, boolean shared=false);
		Promise<void> requestRealityChange(Reality reality);

		long requestFrame(XRFrameRequestCallback callback);
		void cancelFrame(long handle);

		readonly attribute boolean hasStageBounds;
		readonly attribute XRStageBounds? stageBounds;

		Promise<void> end();

		attribute EventHandler onblur;
		attribute EventHandler onfocus;
		attribute EventHandler onresetpose;
		attribute EventHandler onrealitychanged;
		attribute EventHandler onrealityconnect;
		attribute EventHandler onrealitydisconnect;
		attribute EventHandler onboundschange;
		attribute EventHandler onended;
	};

A script that wishes to make use of an XRDisplay can request an XRSession. This session provides a list of the available realities that the script may request as well as make a request for an animation frame.

_The XRSession plays the same basic role as the VRSession, with the addition of Reality management._

	enum XRSessionType { "reality", "augmentation" };

	dictionary XRSessionCreateParametersInit {
		required boolean exclusive = true;
		optional XRSessionType type = "reality";
	};

	interface XRSessionCreateParameters {
		readonly attribute boolean exclusive;
		readonly attribute XRSessionType type;
	};

- 'exclusive' needs to be rethought given the new use of XRDisplay for magic window. Do we still need sessions that just wants sensor data? 

## Reality

	interface Reality : EventTarget {
		readonly atttribute DOMString name;
		readonly attribute boolean isShared; // True if sessions other than the creator can access this Reality
		readonly attribute boolean isPassthrough; // True if the Reality is a view of the outside world, not a full VR

		XRCoordinateSystem? getCoordinateSystem(...XRFrameOfReferenceType type); // Tries the types in order, returning the first match or null if none is found

		attribute EventHandler onchange;
	};

A Reality represents a view of the world, be it the real world via sensors or a virtual world that is rendered with WebGL or WebGPU.

Realities can be shared among XRSessions, with multiple scripts rendering into their separate XRLayer.context that are then composited by the UA with the Reality being rearmost.

- How do we support configuration (e.g. change white balance on camera input, change options on map view)?

## XRPointCloud

	interface XRPointCloud {
		readonly attribute Float32Array points; // Each point is [x, y, z, confidence in range 0-1]
	}

## XRLightEstimate

	interface XRLightEstimate {
		readonly attribute double ambientIntensity;
		readonly attribute double ambientColorTemperature;
	}

- Should we support point and directional light estimates?

## XRAnchor

	interface XRAnchor {
		readonly attribute DOMString uid;
		readonly attribute XRCoordinates coordinates;
	}

XRAnchors provide per-frame coordinates which the Reality attempts to pin "in place". In a virtual Reality these coordinates do not change. In a Reality based on environment mapping sensors, the anchors may change coordinates on a per-frame bases as the system refines its map.

## XRPlaneAnchor

	interface XRPlaneAnchor : XRAnchor {
		readonly attribute double width;
		readonly attribute double length;
	}

XRPlaneAnchors usually represent surfaces like floors, table tops, or walls.

## XRAnchorOffset 

	interface XRAnchorOffset {
		readonly attribute XRAnchor anchor;
		readonly attribute double x;
		readonly attribute double y;
		readonly attribute double z;
	}

XRAnchorOffset represents a position in relation to an anchor, returned when XRPresentationFrame.findAnchor casts a ray. If the ray intersects an XRPlaneAnchor, the XRAnchorOffset returned will contain that anchor and the position in the anchor's coordinate system where the intersection occurred. If the ray passes near an existing XRAnchor, the XRAnchorOffset returned may contain that anchor and the relative position of closest point on the ray. The Reality may, if possible, create a new XRAnchor for use in the XRAnchorOffset or return null if the ray does not intersect anything in the reality or it is not possible to anchor at the intersection.

- How near is "near" for rays passing XRAnchors? ARKit suggests to stay no more than 3M from anchors or suffer lever arm issues.

## XRManifold

	interface XRManifold {
		TBD
	}

- How do we expose the manifold vertices and edges as well as its extent (FOV only, full sphere, etc)?

## XRStageBounds

	interface XRStageBounds {
		readonly attribute XRCoordinates center;
		readonly attribute FrozenArray<XRStageBoundsPoint>? geometry;
	};

## XRStageBoundsPoint 

	interface XRStageBoundsPoint {
		readonly attribute double x;
		readonly attribute double z;
	};

## XRPresentationFrame

	interface XRPresentationFrame {
		readonly attribute FrozenArray<XRView> views;

		readonly attribute boolean hasPointCloud;
		readonly attribute XRPointCloud? pointCloud;

		readonly attribute boolean hasManifold;
		readonly attribute XRManifold? manifold;

		readonly attribute boolean hasLightEstimate;
		readonly attribute XRLightEstimate? lightEstimate;

		readonly attribute sequence<XRAnchor> anchors;
		DOMString addAnchor(XRAnchor anchor);
		void removeAnchor(DOMString uid);
		XRAnchor? getAnchor(DOMString uid);
		XRAnchorOffset? findAnchor(XRCoordinates); // cast a ray to find or create an anchor at the first intersection in the Reality

		XRCoordinateSystem? getCoordinateSystem(...XRFrameOfReferenceType types); // Tries the types in order, returning the first match or null if none is found

		XRViewPose? getViewPose(XRCoordinateSystem coordinateSystem);
	};

_The XRPresentationFrame differs from the VRPresentationFrame with the addition of the point cloud, manifold, light estimates, and anchor management._

- Should we fire an event when a marker or feature based anchor (e.g. a wall, a table top) is detected?
- How can we access camera image buffers aor textures?

## XRView

	interface XRView {
		readonly attribute XREye eye;
		readonly attribute Float32Array projectionMatrix;

		XRViewport? getViewport(XRLayer layer);
	};

## XRViewport

	interface XRViewport {
		readonly attribute long x;
		readonly attribute long y;
		readonly attribute long width;
		readonly attribute long height;
	};


## XRCartographicCoordinates

	enum XRCartographicCoordinatesGeodeticFrame { "WGS84" };

	interface XRCartographicCoordinates {
		attribute XRCartographicCoordinatesGeodeticFrame? geodeticFrame;
		attribute double latitude;
		attribute double longitude;
		attribute double positionAccuracy;
		attribute double altitude;
		attribute double altitudeAccuracy;
		attribute Float32Array orientation; // quaternion x,y,z,w from 0,0,0,1 of East/Up/South 
	}

The XRCartographicCoordinates are used in conjunction with the XRCoordinateSystem to represent a frame of reference that may optionally be positioned in relation to a geodetic frame like WGS84 for Earth, otherwise a sphere is assumed.

- We could find geodetic frames for other planets and moons in this solar system

## XRCoordinateSystem

	enum XRFrameOfReferenceType { "headModel", "eyeLevel", "stage", "geospatial" };

	interface XRCoordinateSystem {
		readonly attribute XRCartographicCoordinates? cartographicCoordinates;
		readonly attribute XRFrameOfReferenceType type;

		Float32Array? getTransformTo(XRCoordinateSystem other);
	};


## XRCoordinates

	interface XRCoordinates {
		readonly attribute XRCoordinateSystem coordinateSystem;
		readonly attribute Float32Array poseMatrix;

		XRCoordinates? getTransformedCoordinates(XRCoordinateSystem otherCoordinateSystem) 
	};


## XRViewPose

	interface XRViewPose {
		readonly attribute Float32Array poseModelMatrix;

		Float32Array getViewMatrix(XRView view);
	};

- Do we need coordinate systems for the poseModelMatrix and getViewMatrix?

## XRLayer

	interface XRLayer : EventTarget {
	};

## XRWebGLLayer

	typedef (WebGLRenderingContext or WebGL2RenderingContext) XRWebGLRenderingContext;

	[Constructor(XRSession session, XRWebGLRenderingContext context, optional XRWebGLLayerInit layerInit)]

	interface XRWebGLLayer : XRLayer {
		readonly attribute XRWebGLRenderingContext context;

		readonly attribute boolean antialias;
		readonly attribute boolean depth;
		readonly attribute boolean stencil;
		readonly attribute boolean alpha;
		readonly attribute boolean multiview;

		readonly attribute WebGLFramebuffer framebuffer;
		readonly attribute long framebufferWidth;
		readonly attribute long framebufferHeight;

		void requestViewportScaling(double viewportScaleFactor);
	};


