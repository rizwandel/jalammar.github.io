---
published: false
---




# Supercharging Android Applications with Machine Learning Using Tensorflow

In November 2015, Google [announced](https://googleblog.blogspot.com/2015/11/tensorflow-smarter-machine-learning-for.html) and open sourced [TensorFlow](https://www.tensorflow.org/), its latest and greatest machine learning system. Needless to say, this was a huge deal for three simple reasons:
1. Machine Learning expertise: Google is probably the dominant force in machine learning. Its prominence in search owes quite a bit to the strides they achieved in machine learning. It employees significant machine learning talents in research and engineering (including distinguished scientists like [Geoffrey Hinton](https://en.wikipedia.org/wiki/Geoffrey_Hinton)).
2. Scalability: the announcement noted that tensorflow was initially designed for internal use and that it's already in production for some live product features.
3. Ability to run on Mobile.

This last reason is the operating reason for us here since this post addresses Android devs. If you examine the [tensorflow repo on GitHub](https://github.com/tensorflow/tensorflow), you'll find a little [tensorflow/examples/android](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/examples/android) directory. I'll try to shed some light on the Android tensorflow example and some of the things going on under the hood.

The Android example page give you an idea on how to build the app, and ultimately culminates in producing this APK (I built and uploaded the APK to save you some time since the building process requires installing the Android NDK and Bazel, Google's build tool).


ِ
## A Look of Recognition
The app glances out through your camera and tries to identify the objects it sees. Sometimes it does a good job, other times it can't quite pin down the object, and at times it leads to thought provoking guesses! Overall, it actually feels quite magical.

![android_tensorflow_classifier_results.jpg]({{site.baseurl}}/_posts/android_tensorflow_classifier_results.jpg)


The app accomplishes this feat using a bundled machine learning model running on the device in tensorflow. The model is pre-trained against millions of images so that it can look at the photos the camera feeds it and classify the object into its best guess (from the 1000 object classifications it knows). Along with its best guess, it shows a confidence score to indicate how sure it is about its guess.

## App Structure Walkthrough

[![android-tensorflow-app-structure.png](https://raw.githubusercontent.com/jalammar/jalammar.github.io/master/_posts/android-tensorflow-app-structure.png)](https://raw.githubusercontent.com/jalammar/jalammar.github.io/master/_posts/android-tensorflow-app-structure.png)

The core TensorFlow engine is built with C++, but programmers can write their TensorFlow software in either C++ or Python. The Android TensorFlow example uses the C++ interface in the following manner:
1. On startup, the app launches an Android activity ([CameraActivity.java](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/examples/android/src/org/tensorflow/demo/CameraActivity.java)) which then starts a fragment ([CameraConnectionFragment.java](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/examples/android/src/org/tensorflow/demo/CameraConnectionFragment.java))
2. The fragment does some setup to basically start the camera and feed the incoming stream of images to an object it instantiates ([TensorflowImageListener.java](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/examples/android/src/org/tensorflow/demo/TensorflowImageListener.java)
3. The listener consults the classifier ([TensorflowClassifier.java](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/examples/android/src/org/tensorflow/demo/TensorflowClassifier.java)) about each image it gets, and returns the classification and confidence score for each image.

The good thing is that most of this logic is in normal Android Java SDK territory. So this should be familiar to most Android devs. You may ask "So where is the C++?".

If you look closely at TensorflowClassifier, you may notice the following three methods:

	public native int initializeTensorflow( );

	private native String classifyImageBmp(Bitmap bitmap);

	private native String classifyImageRgb(int[] output, int width, int height);

The `native` keywords in these method signatures indicate that these methods are implemented in native C++ code. Look for them under the `android/jni` directory and true enough, you'll find [tensorflow_jni.cc](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/examples/android/jni/tensorflow_jni.cc)

	JNIEXPORT jint JNICALL
	TENSORFLOW_METHOD(initializeTensorflow)( ) {
    ...
    }

## The Model
As you read the example's readme.md, you'll notice that it instructs you to download a zip file containing the TensorFlow model and add it to the `assets` directory. This zip file contains two files that are important for us:
1. `tensorflow_inception_graph.pb`- At 54 MBs unzipped, this file consititutes the majority of the APK size (58 MBs). This is our pre-trained machine learning model and where the magic comes from. It's a pre-built TensorFlow GraphDef describing the structure and weights of the network. If you open the file with a text editor, you'll find 3.3 million lines of hex values looking like this: `0a36 0a05 696e 7075`. That's because this GraphDef is serialized and encoded with Google's [Protocol Buffers](https://developers.google.com/protocol-buffers/?hl=en) so it can be deserialized across different platforms (think of it as an encoded JSON file). 
2. `imagenet_comp_graph_label_strings.txt`- this contains the 1000 classifications that the output of the model corresponds to (e.g. "kit fox", "English setter", "Siberian husky"). These classifications are [defined](http://image-net.org/challenges/LSVRC/2014/browse-synsets) by the ImageNet Large Scale Visual Recognition Challenge which the model was built to compete in.

The model here is what's known as a deep convolutional neural network. It is built in the Inception architecture described in [Going Deeper with Convolutions](http://www.cv-foundation.org/openaccess/content_cvpr_2015/papers/Szegedy_Going_Deeper_With_2015_CVPR_paper.pdf). [Convulutional neural networks](https://youtu.be/bEUX_56Lojc?t=2m53s) are some of the most popular models in deep learning. They have been very successful in image recognition (so much so, that most highly ranked teams in the competition used them).

The model is read from the file and fed into tensorflow when the app starts up. This [code](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/examples/android/jni/tensorflow_jni.cc#L50) is actually really interesting to read and see how to communicate with tensorflow (if you run the app with your device connected to your computer, you can see these helpful log messages printed in logcat).

## The Possibilities
Using a pre-trained model in your app seems to be the lowest hanging fruit for mobile TensorFlow apps at the moment.