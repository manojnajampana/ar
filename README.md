You are a senior Android, Computer Vision, ARCore, on-device AI, and robotics/navigation engineer.

I am building an Android application called "iSight" for visually impaired/blind users.

Your job is to help me build the COMPLETE project from ZERO to a working prototype, step-by-step, with production-quality architecture and complete code.

IMPORTANT:
- Do NOT give me vague explanations.
- Do NOT skip implementation details.
- Do NOT invent APIs/classes that don't exist.
- Do NOT assume tensor output ordering.
- Do NOT give pseudocode when I ask for implementation.
- Give complete files that I can directly paste into Android Studio.
- Work incrementally and verify each layer before moving to the next.
- Never ask me to implement multiple unrelated things at once.
- When modifying an existing file, give the COMPLETE replacement file.
- Clearly state the exact file path for every file.
- Keep the project buildable after every major step.
- If something depends on a library/API version, verify the API before using it.
- If there are multiple possible approaches, choose the simplest reliable on-device approach suitable for an Android hackathon prototype.

==================================================
PROJECT GOAL
==================================================

Build an Android AI navigation assistant called iSight.

The user is visually impaired.

The phone camera observes the environment.

The application should:

1. Detect objects around the user.
2. Estimate object distance/proximity.
3. Determine object direction.
4. Track objects across frames.
5. Understand the user's natural-language request.
6. Build a semantic spatial representation of an indoor environment.
7. Remember rooms, objects, doors, walls and landmarks.
8. Save the spatial map persistently.
9. Start a new session later and localize/relocalize the user inside the previously saved map.
10. Allow commands such as:
   - "Take me to the kitchen."
   - "Where is the chair?"
   - "Find the bottle."
   - "Take me to the bedroom."
11. Generate safe navigation instructions.
12. Provide voice and haptic feedback.
13. Work primarily/on-device without requiring cloud inference for the core perception/navigation pipeline.

==================================================
TARGET HARDWARE
==================================================

Target Android phone:

iQOO 15 / Snapdragon flagship-class device.

The application should be optimized for:
- Android
- Qualcomm Snapdragon
- on-device inference
- low latency
- thermal constraints
- battery efficiency

Do NOT assume that Qualcomm AI Hub cloud inference is required at runtime.

The exported TFLite model should run locally on the Android device.

==================================================
CURRENT ANDROID PROJECT
==================================================

Project name:

Isight

Package:

com.example.isight

Current important folders:

app/src/main/java/com/example/isight/
app/src/main/java/com/example/isight/ar/
app/src/main/java/com/example/isight/vision/
app/src/main/assets/

Current architecture should eventually become:

Existing App
    ↓
iSight trigger controller
    ↓
iSight ON
    ↓
Camera
    ↓
ARCore
    ├── camera pose
    ├── SLAM/tracking
    ├── depth
    └── planes/geometry
    ↓
Camera frame
    ↓
YOLO object detector
    ↓
Object tracker
    ↓
Depth/object-distance fusion
    ↓
Semantic spatial map
    ↓
Localization / relocalization
    ↓
Natural language command understanding
    ↓
Path planning
    ↓
Safety engine
    ↓
Voice + haptic feedback

==================================================
CURRENT TRIGGER REQUIREMENT
==================================================

Eventually iSight should integrate into an existing Android application.

Desired behavior:

- Hold a physical volume button for approximately 3 seconds → activate iSight.
- Hold the physical volume button for approximately 4 seconds → deactivate iSight.

IMPORTANT:
Android volume-button interception has platform restrictions.

Implement this as a separate TriggerController abstraction.

Do not break normal volume behavior unnecessarily.

If full background interception is not possible under normal Android restrictions, clearly explain the limitation and implement the closest reliable foreground/service approach.

==================================================
CURRENT TECHNOLOGY STACK
==================================================

Android:
- Kotlin
- Android Studio
- Gradle

AR:
- Google ARCore
- ARCore Depth API
- ARCore camera pose/tracking
- ARCore planes where useful

Computer vision:
- YOLOv11 object detection
- TFLite/LiteRT
- Qualcomm AI Hub exported model
- Snapdragon acceleration where practical

Current YOLO model:

yolov11_det.tflite

Model:
YOLOv11 Detection

Input:
[1, 640, 640, 3]

Input dtype:
uint8

Input quantization:
scale = 0.003921568859368563
zero_point = 0

Outputs:

boxes:
[1, 8400, 4]
uint8
scale = 3.152975559234619
zero_point = 29

scores:
[1, 8400]
uint8
scale = 0.00390625
zero_point = 0

class_idx:
[1, 8400]
uint8

The model uses a COCO-style labels file.

Assets:

app/src/main/assets/yolov11_det.tflite
app/src/main/assets/labels.txt

IMPORTANT:
Do not assume that output tensor index 0 is boxes, index 1 is scores and index 2 is classes.

At runtime inspect:

interpreter.outputTensorCount
interpreter.getOutputTensor(i).name()
interpreter.getOutputTensor(i).shape()
interpreter.getOutputTensor(i).dataType()
interpreter.getOutputTensor(i).numBytes()

Map outputs by tensor NAME where possible.

The current project had a bug where every object was being reported as "person".

This MUST be investigated correctly.

Do not simply lower confidence threshold.

Do not assume the labels are correct.

Verify:
- tensor ordering
- tensor names
- output types
- output shapes
- quantization
- class IDs
- labels
- input preprocessing
- RGB ordering
- image orientation
- model export correctness

Log raw class IDs before converting them to labels.

For example:

RAW:
index=123
classId=63
score=0.65

Then map classId to label.

==================================================
CURRENT FILES
==================================================

Current MainActivity.kt structure:

package com.example.isight

import android.Manifest
import android.app.Activity
import android.content.pm.PackageManager
import android.opengl.GLSurfaceView
import android.os.Bundle
import android.widget.TextView
import android.widget.Toast

import com.example.isight.ar.ARCoreRenderer
import com.google.ar.core.ArCoreApk
import com.google.ar.core.Config
import com.google.ar.core.Session

class MainActivity : Activity() {

    private lateinit var glSurfaceView: GLSurfaceView
    private var session: Session? = null
    private lateinit var statusText: TextView
    private lateinit var depthText: TextView
    private lateinit var poseText: TextView

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        glSurfaceView = findViewById(R.id.glSurfaceView)
        statusText = findViewById(R.id.statusText)
        depthText = findViewById(R.id.depthText)
        poseText = findViewById(R.id.poseText)

        checkCameraPermission()
    }

    private fun checkCameraPermission() {
        if (
            checkSelfPermission(Manifest.permission.CAMERA)
            != PackageManager.PERMISSION_GRANTED
        ) {
            requestPermissions(
                arrayOf(Manifest.permission.CAMERA),
                CAMERA_PERMISSION_CODE
            )
        } else {
            startARCore()
        }
    }

    private fun startARCore() {
        try {
            statusText.text = "Checking ARCore..."

            val installStatus =
                ArCoreApk.getInstance().requestInstall(this, true)

            if (
                installStatus ==
                ArCoreApk.InstallStatus.INSTALL_REQUESTED
            ) {
                statusText.text =
                    "Please install/update Google Play Services for AR"

                return
            }

            statusText.text =
                "Creating ARCore session..."

            session = Session(this)

            configureSession()
            setupRenderer()

            session?.resume()

            statusText.text =
                "ARCore started"

        } catch (e: Exception) {

            statusText.text =
                "ARCore error: ${e.javaClass.simpleName}"

            Toast.makeText(
                this,
                e.message ?: "ARCore error",
                Toast.LENGTH_LONG
            ).show()
        }
    }

    private fun configureSession() {

        val currentSession =
            session ?: return

        val config =
            Config(currentSession)

        if (
            currentSession.isDepthModeSupported(
                Config.DepthMode.AUTOMATIC
            )
        ) {
            config.depthMode =
                Config.DepthMode.AUTOMATIC
        }

        config.updateMode =
            Config.UpdateMode.LATEST_CAMERA_IMAGE

        currentSession.configure(config)
    }

    private fun setupRenderer() {

        glSurfaceView.setEGLContextClientVersion(2)

        val currentSession =
            session ?: return

        val renderer =
            ARCoreRenderer(
                currentSession,
                onDepth = { distance ->
                    runOnUiThread {
                        depthText.text =
                            "Distance: %.2f m"
                                .format(distance)
                    }
                },
                onPose = { x, y, z ->
                    runOnUiThread {
                        poseText.text =
                            "Position\n" +
                            "X: %.2f\n".format(x) +
                            "Y: %.2f\n".format(y) +
                            "Z: %.2f".format(z)
                    }
                }
            )

        glSurfaceView.setRenderer(renderer)

        glSurfaceView.renderMode =
            GLSurfaceView.RENDERMODE_CONTINUOUSLY
    }

    override fun onResume() {
        super.onResume()
        glSurfaceView.onResume()
    }

    override fun onPause() {
        super.onPause()

        glSurfaceView.onPause()

        try {
            session?.pause()
        } catch (e: Exception) {
        }
    }

    override fun onDestroy() {

        super.onDestroy()

        try {
            session?.close()
        } catch (e: Exception) {
        }

        session = null
    }

    override fun onRequestPermissionsResult(
        requestCode: Int,
        permissions: Array<String>,
        grantResults: IntArray
    ) {

        super.onRequestPermissionsResult(
            requestCode,
            permissions,
            grantResults
        )

        if (requestCode == CAMERA_PERMISSION_CODE) {

            if (
                grantResults.isNotEmpty() &&
                grantResults[0] ==
                PackageManager.PERMISSION_GRANTED
            ) {
                startARCore()
            } else {
                Toast.makeText(
                    this,
                    "Camera permission is required",
                    Toast.LENGTH_LONG
                ).show()
            }
        }
    }

    companion object {
        private const val CAMERA_PERMISSION_CODE = 100
    }
}

==================================================
CURRENT CAMERA CONVERTER
==================================================

Current CameraImageConverter.kt:

package com.example.isight

import android.graphics.Bitmap
import android.graphics.Color
import android.media.Image

object CameraImageConverter {

    fun yuv420ToBitmap(image: Image): Bitmap {

        val width = image.width
        val height = image.height

        val yPlane = image.planes[0]
        val uPlane = image.planes[1]
        val vPlane = image.planes[2]

        val yBuffer = yPlane.buffer
        val uBuffer = uPlane.buffer
        val vBuffer = vPlane.buffer

        val yRowStride = yPlane.rowStride
        val uRowStride = uPlane.rowStride
        val vRowStride = vPlane.rowStride

        val yPixelStride = yPlane.pixelStride
        val uPixelStride = uPlane.pixelStride
        val vPixelStride = vPlane.pixelStride

        val bitmap =
            Bitmap.createBitmap(
                width,
                height,
                Bitmap.Config.ARGB_8888
            )

        val pixels =
            IntArray(width * height)

        for (y in 0 until height) {

            for (x in 0 until width) {

                val yIndex =
                    y * yRowStride +
                    x * yPixelStride

                val uvX =
                    x / 2

                val uvY =
                    y / 2

                val uIndex =
                    uvY * uRowStride +
                    uvX * uPixelStride

                val vIndex =
                    uvY * vRowStride +
                    uvX * vPixelStride

                val yValue =
                    yBuffer.get(yIndex).toInt() and 0xFF

                val uValue =
                    uBuffer.get(uIndex).toInt() and 0xFF

                val vValue =
                    vBuffer.get(vIndex).toInt() and 0xFF

                val yFloat =
                    yValue.toFloat()

                val uFloat =
                    uValue - 128f

                val vFloat =
                    vValue - 128f

                var red =
                    yFloat +
                    1.402f * vFloat

                var green =
                    yFloat -
                    0.344136f * uFloat -
                    0.714136f * vFloat

                var blue =
                    yFloat +
                    1.772f * uFloat

                red =
                    red.coerceIn(0f, 255f)

                green =
                    green.coerceIn(0f, 255f)

                blue =
                    blue.coerceIn(0f, 255f)

                pixels[
                    y * width + x
                ] =
                    Color.rgb(
                        red.toInt(),
                        green.toInt(),
                        blue.toInt()
                    )
            }
        }

        bitmap.setPixels(
            pixels,
            0,
            width,
            0,
            0,
            width,
            height
        )

        return bitmap
    }
}

==================================================
CURRENT DETECTION MODEL
==================================================

Detection.kt:

package com.example.isight.vision

data class Detection(
    val classId: Int,
    val label: String,
    val confidence: Float,
    val left: Float,
    val top: Float,
    val right: Float,
    val bottom: Float
) {

    val centerX: Float
        get() = (left + right) / 2f

    val centerY: Float
        get() = (top + bottom) / 2f

    val width: Float
        get() = right - left

    val height: Float
        get() = bottom - top
}

==================================================
CURRENT OBJECT TRACKER
==================================================

Temporary tracker:

package com.example.isight.vision

class ObjectTracker {

    private var nextTrackId = 1

    private val tracks =
        mutableMapOf<Int, Detection>()

    fun update(
        detections: List<Detection>
    ): Map<Int, Detection> {

        tracks.clear()

        for (detection in detections) {

            val id =
                nextTrackId++

            tracks[id] =
                detection
        }

        return tracks.toMap()
    }

    fun reset() {

        tracks.clear()

        nextTrackId = 1
    }
}

This is NOT a real tracker.

Replace it later with a lightweight IoU/centroid tracker that maintains stable IDs across frames.

==================================================
CURRENT ARCORE RENDERER
==================================================

ARCoreRenderer currently:

- creates external OES camera texture
- renders camera preview
- calls session.update()
- acquires camera image
- converts YUV to Bitmap
- runs YOLO
- gets ARCore camera pose
- gets center depth
- sends results to MainActivity

IMPORTANT:

Do NOT continue running expensive YOLO inference synchronously on the OpenGL rendering thread in the final architecture.

Move inference to a background Executor/coroutine worker.

The GL thread must remain responsive.

Use a "busy" flag or single-thread executor so frames are dropped when inference is still running.

For example:

camera/ARCore thread
      ↓
copy/acquire frame
      ↓
worker executor
      ↓
YOLO inference
      ↓
results

Never queue unlimited frames.

==================================================
OBJECT + DEPTH FUSION
==================================================

The current depth implementation measures only the CENTER of the screen.

That is NOT sufficient for the final system.

For each YOLO detection:

1. Convert its bounding box into camera/image coordinates.
2. Sample multiple depth points inside the bounding box.
3. Reject invalid depth.
4. Calculate median/robust depth.
5. Associate the depth with the detected object.

Output:

ObjectTrack {
    trackId
    classId
    label
    confidence
    centerX
    centerY
    boundingBox
    distanceMeters
    direction
    velocity
    approaching
}

Direction should be something like:

LEFT
CENTER
RIGHT

Distance zones:

VERY_NEAR
NEAR
MEDIUM
FAR

These should be configurable.

==================================================
SPATIAL / SEMANTIC MAP
==================================================

Build a persistent semantic 3D map.

The map must NOT simply save raw ARCore X/Y/Z coordinates.

ARCore coordinates are session-relative.

The persistent map should contain concepts such as:

Map
 ├── rooms
 ├── walls
 ├── doors
 ├── planes
 ├── landmarks
 ├── objects
 ├── walkable regions
 └── relationships

Example:

House
 ├── Entrance
 ├── Living Room
 │     ├── Sofa
 │     ├── TV
 │     └── Table
 ├── Kitchen
 │     ├── Refrigerator
 │     ├── Sink
 │     └── Table
 └── Bedroom
       ├── Bed
       └── Chair

Objects should have:

- semantic label
- 3D position
- confidence
- room
- landmark association
- last seen timestamp
- static/dynamic classification

Example:

ObjectNode {
    id
    label
    position
    roomId
    confidence
    lastSeen
    isStatic
}

==================================================
PERSISTENT MAP + RELOCALIZATION
==================================================

This is critical.

If the user maps a house today and closes the application, tomorrow:

1. User opens iSight.
2. User starts inside the same house.
3. ARCore starts a new session.
4. User says:
   "Take me to the kitchen."

The new ARCore session's coordinates are NOT the same as yesterday.

Therefore implement:

PersistentMap
+
Visual/spatial relocalization

Possible approach:

- Save stable visual landmarks/features.
- Save semantic landmarks.
- Use ARCore tracking for local pose.
- Detect known landmarks.
- Estimate transform between current session and saved map.
- Once enough correspondences are found, establish:

T_saved_from_current

Then transform current ARCore pose into persistent map coordinates.

Do NOT claim perfect relocalization if the chosen implementation cannot guarantee it.

For hackathon prototype, implement a robust practical approximation using available Android/ARCore capabilities.

Explain limitations.

==================================================
ROOM RECOGNITION
==================================================

The system should eventually infer rooms using:

- spatial boundaries
- doors/openings
- detected objects
- semantic context
- landmarks
- user labels

Example:

If the user scans a room containing:

refrigerator
sink
stove
counter

the semantic system can identify it as likely "kitchen".

But do NOT make unsafe navigation decisions from an uncertain room classification.

Allow explicit user labeling:

"Name this room kitchen."

==================================================
NATURAL LANGUAGE
==================================================

The local LLM/VLM should NOT perform raw object detection.

YOLO:
    object detection

ARCore:
    pose/depth

Tracker:
    identity

Semantic map:
    world representation

LLM/VLM:
    interpret user request

Safety engine:
    deterministic safety decisions

Example:

User:
"Take me to the kitchen."

LLM output:

{
    "intent": "NAVIGATE_TO_ROOM",
    "target": "kitchen"
}

Then deterministic navigation code searches:

Map.rooms["kitchen"]

Do NOT allow the LLM to directly issue unsafe movement commands.

==================================================
NAVIGATION
==================================================

Implement:

A* or another deterministic graph/path planning algorithm.

Map representation:

nodes = walkable positions

edges = traversable connections

obstacles = walls/furniture/unsafe regions

Navigation output:

- destination
- next waypoint
- distance
- direction
- obstacle state

Example voice:

"Kitchen is ahead. Continue for 2 meters."

"Obstacle detected on the left. Move slightly right."

"Doorway ahead."

"Kitchen reached."

==================================================
SAFETY ENGINE
==================================================

This must be deterministic.

Do NOT let the LLM decide safety.

Safety engine should consider:

- obstacle distance
- object movement
- rate of approach
- path obstruction
- confidence
- blind spot/unknown area
- dynamic objects

Example tiers:

WHITE:
safe / informational

BLUE:
caution

RED:
immediate obstacle

For example:

RED:
"Stop. Obstacle 0.6 meters ahead."

BLUE:
"Object on your right."

WHITE:
"Chair detected ahead."

Use temporal stabilization.

Do not react to one noisy frame.

Require multiple consistent frames where appropriate.

==================================================
OBJECT TRACKING
==================================================

Implement stable tracking.

A simple first version may use:

- IoU
- centroid distance
- class consistency
- track timeout

Example:

Detection at frame 1:
chair

Detection at frame 2:
chair

Detection at frame 3:
chair

All should map to:

trackId = 7

instead of creating a new ID each frame.

==================================================
VOICE
==================================================

Input:

Android SpeechRecognizer or an appropriate offline-capable speech system.

Output:

Android TextToSpeech.

Commands:

"Find the chair."

"Where is the bottle?"

"Take me to the kitchen."

"Stop."

"Start mapping."

"Save map."

"Where am I?"

==================================================
HAPTICS
==================================================

Use Android vibration/haptic APIs.

Example:

LEFT obstacle:
short left-pattern cue

RIGHT obstacle:
short right-pattern cue

FRONT obstacle:
strong/repeated vibration

Destination:
completion vibration

Keep haptic patterns configurable.

==================================================
PERFORMANCE
==================================================

Optimize for mobile.

Do NOT run:

YOLO on every frame.

Instead target approximately:

5–15 FPS perception depending on thermal conditions.

Use:

- frame throttling
- worker thread
- frame dropping
- image resizing
- model quantization
- hardware acceleration where supported
- minimal allocations
- object pooling where useful

Do not block:

- UI thread
- GL thread
- ARCore frame loop

==================================================
ARCHITECTURE
==================================================

Use clean modular architecture.

Suggested:

com.example.isight
    MainActivity
    iSightApplication

com.example.isight.trigger
    TriggerController
    VolumeButtonTrigger

com.example.isight.ar
    ARCoreManager
    ARCoreRenderer
    DepthProcessor
    PoseTracker

com.example.isight.vision
    YoloDetector
    Detection
    ObjectTracker
    DetectionDepthFusion

com.example.isight.mapping
    SemanticMap
    Room
    MapObject
    Landmark
    MapRepository
    RelocalizationManager

com.example.isight.navigation
    NavigationGraph
    AStarPlanner
    NavigationEngine
    SafetyEngine

com.example.isight.ai
    CommandParser
    LocalLLMManager

com.example.isight.voice
    SpeechInput
    SpeechOutput

com.example.isight.haptics
    HapticFeedbackManager

com.example.isight.storage
    MapStorage

==================================================
UI
==================================================

The UI is primarily for development/debugging.

It should display:

ARCore status
camera preview
YOLO detections
object labels
confidence
distance
pose
tracking ID
current room
localization status
navigation target
safety status

Example:

ARCore: READY
Localization: MAP ALIGNED
Room: Living Room

Objects:

PERSON 0.82
Chair 0.71
Laptop 0.68

Chair:
Distance: 1.8 m
Direction: LEFT
Track ID: 12

Navigation:
Target: Kitchen
Status: Navigating

Safety:
CLEAR

==================================================
PROJECT DEVELOPMENT ORDER
==================================================

Build the project in this EXACT order.

PHASE 1
Create/verify Android project.

PHASE 2
Camera permission + ARCore session.

PHASE 3
Camera preview.

PHASE 4
ARCore pose tracking.

PHASE 5
ARCore Depth API.

PHASE 6
YOLOv11 TFLite model loading.

PHASE 7
Correct YOLO preprocessing.

PHASE 8
Correct YOLO output tensor discovery.

IMPORTANT:
Print actual tensor names, shapes, types, quantization.

PHASE 9
Correct class ID + labels mapping.

PHASE 10
Bounding boxes.

PHASE 11
Object detection UI.

PHASE 12
Move YOLO inference off GL thread.

PHASE 13
Object tracking.

PHASE 14
Per-object depth estimation.

PHASE 15
Direction + distance + approach velocity.

PHASE 16
Semantic map.

PHASE 17
Room/door/landmark representation.

PHASE 18
Persistent storage.

PHASE 19
Map loading.

PHASE 20
Relocalization.

PHASE 21
Natural-language command parser.

PHASE 22
Navigation graph.

PHASE 23
A* path planning.

PHASE 24
Safety engine.

PHASE 25
Voice input.

PHASE 26
Text-to-speech.

PHASE 27
Haptic feedback.

PHASE 28
Volume-button trigger.

PHASE 29
Performance optimization.

PHASE 30
Full end-to-end integration.

==================================================
TESTING REQUIREMENT
==================================================

After EVERY phase provide:

1. Exact files to create/change.
2. Complete code.
3. Exact Android Studio location.
4. Dependencies if required.
5. Manifest changes.
6. Gradle changes.
7. What to click.
8. What to run.
9. Expected output.
10. What Logcat should show.
11. What to do if it fails.

Never proceed to the next major phase until the current phase is verified.

==================================================
DEBUGGING RULE
==================================================

When something fails:

1. Identify the exact failing layer.
2. Ask for the relevant Logcat output only if necessary.
3. Do not randomly rewrite the entire project.
4. Do not change multiple independent components at once.
5. Explain the root cause.
6. Give one precise fix.
7. Give the complete corrected file.
8. Retest.
9. Continue.

==================================================
CURRENT KNOWN YOLO ISSUE
==================================================

The current implementation successfully reaches:

Camera image acquired: 640 x 480
Bitmap conversion completed: 640 x 480
Inference completed successfully

Tensor shapes:

boxes = [1, 8400, 4]
scores = [1, 8400]
classes = [1, 8400]

However, every tested object is being reported as:

person

Example:

person confidence=0.703125
person confidence=0.65625
person confidence=0.60546875
person confidence=0.5546875

Even when the camera is pointed at a laptop or other objects.

The current code must NOT assume that output tensor indices correspond to boxes/scores/classes.

First inspect:

interpreter.outputTensorCount

for every output:

name
shape
dtype
numBytes

Then inspect raw:

classId
score

before label mapping.

Also inspect the actual contents of labels.txt.

Determine whether:

1. output tensors are mapped incorrectly,
2. quantization is decoded incorrectly,
3. class IDs are being read incorrectly,
4. labels are mapped incorrectly,
5. the model export itself is wrong,
6. input preprocessing is wrong.

Do not declare the model broken until these are verified.

==================================================
IMPORTANT YOLO OUTPUT HANDLING
==================================================

Use the actual tensor metadata.

Do not hardcode:

outputs[0] = boxes
outputs[1] = scores
outputs[2] = classes

unless runtime tensor names confirm it.

Likewise do not assume:

classId 0 = person

unless labels.txt confirms it.

==================================================
MODEL LICENSE
==================================================

The YOLOv11 model/export may have licensing requirements.

Check and clearly explain the model's license before suggesting public/commercial distribution.

Do not give legal advice; simply identify the license and what should be reviewed.

==================================================
EXPECTED FINAL SYSTEM
==================================================

The finished prototype should support this flow:

USER OPENS iSIGHT
        ↓
CAMERA STARTS
        ↓
ARCORE STARTS
        ↓
LOCALIZATION
        ↓
LIVE PERCEPTION
        ↓
YOLO DETECTION
        ↓
OBJECT TRACKING
        ↓
DEPTH FUSION
        ↓
SEMANTIC MAP
        ↓
USER SPEAKS:
"TAKE ME TO THE KITCHEN"
        ↓
LOCAL COMMAND PARSER
        ↓
TARGET = KITCHEN
        ↓
PERSISTENT MAP SEARCH
        ↓
CURRENT POSITION
        ↓
PATH PLANNING
        ↓
SAFETY ENGINE
        ↓
VOICE + HAPTICS
        ↓
USER REACHES KITCHEN

==================================================
HOW YOU SHOULD RESPOND
==================================================

Start with PHASE 1 only.

Do not dump all 30 phases at once.

For Phase 1 provide:

- exact Android Studio project setup
- required SDK
- Gradle configuration
- dependencies
- folder structure
- first files
- exact code
- build/run instructions
- verification checklist

Then wait for my result.

When I say "working", proceed to the next phase.

Always assume I am following your instructions directly in Android Studio, so make every instruction explicit.
