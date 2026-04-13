# body_detection

A flutter plugin that uses MLKit on iOS/Android platforms to enable body pose and mask detection using Pose Detection and Selfie Segmentation APIs for both static images and live camera stream. When using live camera detection it runs both the camera image aquisition and detectors on the native side, which makes it faster than using the flutter camera plugin separately.

## Features

* Body pose detection.
* Body mask detection (selfie segmentation).
* Run on single image or live camera feed.
* Both camera image aquisition and detectors run together on the native side which makes it faster than using separate plugins for both and passing data between them.
* Uses MLKit for detectors so shares its advantages and disadvantages.

## Installation

### iOS

Minimum required iOS version is 10.0, so if compiling for a lower target make sure to check programatically the iOS version before calling this library's functions.

To be able to use the camera-based detection you need to add a row to the `ios/Runner/Info.plist` file using Xcode:

* `Privacy - Camera Usage Description` and set a usage description message.

If editing the file in plain-text add the following:

```xml
<key>NSCameraUsageDescription</key>
<string>Can I use the camera please?</string>
```

### Android

Set the minimum Android SDK version to 21 (or higher) in your `android/app/build.gradle` file:

```
android {
    defaultConfig {
        minSdkVersion 21
    }
}
```

To use the camera you need to add uses-permission declaration to your `android/app/src/main/AndroidManifest.xml` manifest file:

```
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="YOUR_PACKAGE_NAME">
  <uses-permission android:name="android.permission.CAMERA"/>
  ...
```

## Usage

The plugin has two modes, you can either run a pose or body mask detection on a single image, or start a camera stream and enable or disable either of the detectors while it's running.

### Single Image Detection

The simplest use case is to detect pose or body mask on a single image. The plugin accepts a PNG-encoded image and sends it to the native side, where it uses Google MLKit's Vision APIs to do the hard work. After that, the results are passed back to flutter:

```dart
import 'package:body_detection/body_detection.dart';
import 'package:body_detection/png_image.dart';

PngImage pngImage = PngImage.from(bytes, width: width, height: height);
final pose = await BodyDetection.detectPose(image: pngImage);
```

This plugin provides an extension to flutter's Image widget that converts it to the PNG format so you can use any image source that the widget supports as an input: local asset, network, memory, etc.

```dart
import 'package:body_detection/body_detection.dart';
import 'package:body_detection/png_image.dart';
import 'package:flutter/widgets.dart';

void detectImagePose(Image source) async {
  PngImage? pngImage = await source.toPngImage();
  if (pngImage == null) return;
  final pose = await BodyDetection.detectPose(image: pngImage);
  ...
}
```

If you need the size of the encoded PNG image for your layout purposes (for example the image aspect ratio), you can get that from the returned PngImage object:

```dart
import 'package:body_detection/png_image.dart';
import 'package:flutter/widgets.dart';

Image source;
PngImage? pngImage = await source.toPngImage();
final imageSize = pngImage != null
  ? Size(pngImage!.width.toDouble(), pngImage!.height.toDouble())
  : Size.zero;
```

Pose object returned from the BodyDetection.detectPose call holds a list of landmarks returned from the MLKit detector. Each Landmark has a structure that follows that established by MLKit:

```dart
final pose = await BodyDetection.detectPose(image: pngImage);
for (final landmark in pose!.landmarks) {
  // The detector estimate of how likely it is that the landmark is within the image's frame.
  double inFrameLikelihood = landmark.inFrameLikelihood;

  // Position of the landmark in image plane coordinates with z value estimated by the detector.
  Point3d position = landmark.position;

  // One of the 33 detectable body landmarks.
  PoseLandmarkType type = landmark.type;
}
```

You can draw the pose using a custom painter:

```dart
class PosePainter extends CustomPainter {
  PosePainter({
    required this.pose,
    required this.imageSize,
  });

  final Pose pose;
  final Size imageSize;
  final circlePaint = Paint()..color = const Color.fromRGBO(0, 255, 0, 0.8);
  final linePaint = Paint()
    ..color = const Color.fromRGBO(255, 0, 0, 0.8)
    ..strokeWidth = 2;

  @override
  void paint(Canvas canvas, Size size) {
    final double hRatio =
        imageSize.width == 0 ? 1 : size.width / imageSize.width;
    final double vRatio =
        imageSize.height == 0 ? 1 : size.height / imageSize.height;

    offsetForPart(PoseLandmark part) =>
        Offset(part.position.x * hRatio, part.position.y * vRatio);

    for (final part in pose.landmarks) {
      // Draw a circular indicator for the landmark.
      canvas.drawCircle(offsetForPart(part), 5, circlePaint);

      // Draw text label for the landmark.
      TextSpan span = TextSpan(
        text: part.type.toString().substring(16),
        style: const TextStyle(
          color: Color.fromRGBO(0, 128, 255, 1),
          fontSize: 10,
        ),
      );
      TextPainter tp = TextPainter(text: span, textAlign: TextAlign.left);
      tp.textDirection = TextDirection.ltr;
      tp.layout();
      tp.paint(canvas, offsetForPart(part));
    }

    // Draw connections between the landmarks.
    final landmarksByType = {for (final it in pose.landmarks) it.type: it};
    for (final connection in connections) {
      final point1 = offsetForPart(landmarksByType[connection[0]]!);
      final point2 = offsetForPart(landmarksByType[connection[1]]!);
      canvas.drawLine(point1, point2, linePaint);
    }
  }

  ...
}
```

And use that in your widget tree:

```dart
@override
Widget build(BuildContext context) {
  // Use ClipRect so that custom painter doesn't draw outside of the widget area.
  return ClipRect(
    child: CustomPaint(
      child: _sourceImage,
      foregroundPainter: PosePainter(
        pose: _detectedPose,
        imageSize: _imageSize,
      ),
    ),
  );
}
```

For details see [Google MLKit Pose Detection API documentation](https://developers.google.com/ml-kit/vision/pose-detection).

### Camera Feed Detection

In addition to single image detection, this plugin supports real-time detection of either or both the pose and body mask from camera feed. Both the camera image acquisition and detection run on the native side, so we eliminate the cost of data serialization we would need if we used a separate flutter plugins for each.

To start and stop the camera stream, use the following methods while passing callbacks for when the camera frame or detection results are ready:

```dart
await BodyDetection.startCameraStream(
  onFrameAvailable: (ImageResult image) {},
  onPoseAvailable: (Pose? pose) {},
);
await BodyDetection.stopCameraStream();
```

Detectors are disabled by default. To enable or disable particular detector use the following methods:

```dart
await BodyDetection.enablePoseDetection();
await BodyDetection.disablePoseDetection();
```

You can run those both before starting the camera stream as well as while the stream is already running.

To display camera image you can create an Image widget from bytes:

```dart
void _handleCameraImage(ImageResult result) {
  // Ignore callback if navigated out of the page.
  if (!mounted) return;

  // To avoid a memory leak issue.
  // https://github.com/flutter/flutter/issues/60160
  PaintingBinding.instance?.imageCache?.clear();
  PaintingBinding.instance?.imageCache?.clearLiveImages();

  final image = Image.memory(
    result.bytes,
    gaplessPlayback: true,
    fit: BoxFit.contain,
  );

  setState(() {
    _cameraImage = image;
    _imageSize = result.size;
  });
}
```

ImageResult's `bytes` field contains a JPEG-encoded frame from the camera which Image.memory() factory constructor accepts as its input.

## Example App

#### Running Pose Detection
![Demo](https://github.com/0x48lab/flutter_body_detection/raw/main/example/screenshots/body_pose.png)

#### Running Body Mask Detection
![Demo](https://github.com/0x48lab/flutter_body_detection/raw/main/example/screenshots/body_mask.png)

## Future Developments

This library encodes images from the camera and serializes them when passing them over to flutter. This enables easy access to the image data so one can do additional processing, save frames to storage or send them for over the network, but also adds an overhead which could be avoided if it only passed a texture id which could then be displayed using the Texture widget in the same vein as what camera plugin does.

It does not yet provide any configuration options and rely on defaults ("accurate" pose detection model, raw size mask for selfie segmentation, using first available front camera, default camera image size, JPEG encoding quality set to 60). It would be good to let users choose those settings to match their use case.

Exception handling is not unified and could use some more work.

## Notice

This library is not yet battle-tested. As it's an early release it might have its features and API changed. If you encounter a problem or would like to request a feature, feel free to [open an issue](https://github.com/0x48lab/flutter_body_detection/issues) at the project's GitHub page.

MLKit's Pose Detection and Selfie Segmentation are still in beta and hence this software should also be considered as such.
