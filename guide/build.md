# Building a Sensor
![Picture of board](./media/boards.jpg)

My general plan of attack for the project was:
-	Build an initial dataset with sounds from the Internet
-	Train a basic model
-	Test how well it works on the bench
-	Build a deployable sensor
-	Evaluate general performance and collect more training data
-	Collect detailed performance results

Following that general outline, I will cover how to build a dataset, training a model, deploying it, building a sensor and evaluating model performance.

## Building an initial dataset

I was able to construct an initial dataset by finding labeled audio on the internet. The dataset should have 2 classes of audio, sirens and no sirens. Microsoft has a dataset of speech, called the Microsoft Scalable Noisy Speech Dataset, which also includes a wide selection of environmental noise. I was tipped off to this dataset from Dan Situnayake's [Bird Audio dataset creator](https://github.com/edgeimpulse/bird-data-download) which makes use of this dataset. The noise was recorded at 16kHz, which helped decide the maximum sample rate I would be using.

For the siren sounds, I used the [Sounds of New York City dataset](https://wp.nyu.edu/sonyc/). The samples are labeled for the different classes of sounds that are present in it. You can download the latest version of the dataset [here](https://zenodo.org/record/3966543#.YCw-My2cbUY). Each of the samples is 10 seconds long. To pull out the samples with the sirens in them, I simply sorted the annotation file and moved the appropriate samples into a new directory. 

## Building and Training a Model

When I first got started working with TinyML, I worked directly with the TF framework. While it is definitely doable, it is not that enjoyable. Simple tasks, especially around the pre-processing of data, always ended up being more complex than expected. 

Luckily good tools have started to emerge, that are specifically designed for running ML on Edge devices. 
[Edge Impulse](https://www.edgeimpulse.com) (EI) provides a complete platform that:
-	Helps you collect data
-	Turns data into datasets
-	Includes signal processing and data augmentation
-	Model customization and training
-	Sample code for running models on different embedded platforms

Needless to say, I went with Edge Impulse for this project. While the TensorFlow framework includes a lot of built-in functions for working with images, there is really nothing included to help with audio. The signal processing and feature extraction tools for audio that Edge Impulse includes in their platform saved me a lot of time. 

EI has a tutorial on doing Sound Classification with their platform: https://docs.edgeimpulse.com/docs/audio-classification

I am not going to repeat all the good info in there, so go check that out.

### Uploading Data
The first step is to get your data into EI. They have tools that let you directly acquire data samples from a device. This lets you capture data right off the sensors and upload it. I was working with pre-recorded audio, so I instead used their command line tool for uploading data. To do that you specify a directory with the files and a label for the type of data. You can also specify if the data should be used for Training or Testing.

```bash
$ edge-impulse-uploader --label noise path/to/a/file.wav
```

The initial data I used was not that well balanced, there was a lot more Noise samples than Siren samples, however it was good enough to get started.

### Data Processing
One of the first configurations you have to decide on is how to chop up the audio samples into chunks, referred to as Windows, that can be feed into the ML model. If the full recordings were used, the model would have to be extra-large to be able to take it all in. Since you are going to be chopping up your recordings, it is important that there sound is present throughout the recording. Suppose you have a 10 second recording and a siren is present for only the first 4 seconds. When it gets chopped up into 2 second chunks there will be a number that are labeled as being a Siren but don’t have that sound. To prevent this, I went through all of the siren recordings and got rid of the ones that didn’t have a siren throughout.

I experimented using 1 second and 2 second Windows. I noticed better performance with 2 second Windows. I think this is because sirens have patterns that vary over time. Using a 2 second window helps ensure that the full pattern is captured. Sirens are also a continual sound, so a 2 second window may also help differentiate them from more instantaneous bleeps, bloops and whoops that are not sirens.

EI includes a number processing blocks that are used to turn raw audio into features that you can train an ML model on. For audio processing, they provide Mel Frequency Cepstral Coefficients (MFCC), Mel-filterbank energy (MFE) and Spectrograms. When I first started out, only MFCC was available. It provided good accuracy and allowed for a small model, so I have stuck with it. You  can learn more about how MFCC works here: https://medium.com/prathena/the-dummys-guide-to-mfcc-aceab2450fd

There are also a number of filters you can apply before you transform your audio. The filters can remove parts of the audio spectrum outside of the frequency range of the targeted sound. EI has recently added a great feature which lets you preview the latency and memory requirements of different filter and processing configurations. Without this, you have to train a model and deploy it on device to find this out. If you are looking to continuously capture and classify sound, you have to make sure your processing latency stays low.

In EI interface you get a visual representation of the processed and transformed audio. The ML models you will be training are similar to image classification models and they treat the transformed audio like an image. If you are able to spot the difference between the different classes of audio in this preview, there is a good chance a model will be able to also learn those differences. Since these previews are dynamically generated, you can adjust the parameters to help make those difference more pronounced... while still keeping an eye on the on-device performance.

### Building a Dataset
Once you have configured your processing block, it is time to run your raw audio through it. This will generate extract features from each audio sample. These features, along with the label for the sample, are what you use to train your ML model. All of this processing happens on EI servers, so you do not have to worry about configuring any Python libraries. 

### Configuring a Neural Network
Now that you have a training dataset, it is time to configure a neural network to train. EI has an easy  to use interface that lets you add layers to a network. I was getting some overfitting, where the model was doing really well against the training data, but not as well against the testing data. This was probably because it had started adapt too closely to the training data. I was able to drag in some Dropout layers and this helped correct it. These layers lightly scramble things and help prevent the model from learning the training data to closely.

As the model is training, you get updates on its performance. You can get a sense of how many training cycles are needed by looking for when the performance gains start to plateau. I generally saw that plateau around 40 training cycles or so. EI will automatically pick the cycle that had the highest accuracy, so it doesn’t hurt to have extra training cycles… it just takes longer. When the model has finished training, you are presented with a number of results. Different versions of the model are compiled, so you can compare the impact of quantization on performance. I generally saw almost no difference between the quantized model, where floating point numbers are convert to integers, and the original model. This is great, because the quantized model is much smaller and easier for the MCU to handle. To help evaluate performance, you are given accuracy predictions and a confusion matrix. There is also an interactive visualization of how different samples from the data set were classified. You can click on the incorrectly classified samples and listen to them. This can help you identify labeling errors you may have in your dataset.

After you have your model, you can export it either as a library or as firmware that can load directly onto supported boards. As described in the Design Considerations section, I chose to use Arduino for this project. When you export an Arduino Library from EI, you get a zip file to download. From the Arduino IDE, you can import the library. As with many libraries, it comes with an example Arduino project, making it easy to try your model out on real hardware. 

### Benchtop Evaluation
To get a sense of whether my model worked in the real world, I loaded it up onto a [Arduino Nano 33 BLE Sense](https://www.arduino.cc/en/Guide/NANO33BLESense). This board is made by Arduino, so its support within the Arduino IDE has been pretty thoroughly tested. It is also officially supported by EI. I kept the evaluation simple and pulled up some Youtube clips of sirens on my laptop. Amazingly, using salvaged audio off the internet, I was able to build a working siren detector. Of course, the real question is how it would perform in a real-world environment, against real sirens.

## Building a Sensor
![Lid view](./media/lid.jpg)

While it was amazing to be able to run an ML model on a low-power MCU, being able to run it in the real world was key. Doing this meant packaging the electronics up so they could survive the elements, making sure it has enough power and finding a way to measure performance. While the Arduino Nano 33 BLE Sense is a nice board, it is sort of limited when it comes to expandability. It has a lot of built-in in sensors, but there are not many boards available to add functionality. I wanted to add the following functionality:

-	**Battery power/charging** – It is possible to power the Arduino board by connecting it to a USB battery, but it is much nicer to has a JST connection so you can directly connect a LiPo battery.
-	**SD/MicroSD storage** – This would allow for audio recordings to be saved.
-	**LoRa radio** – This would allow for status and detection information to be remotely monitored. This could also be done using cellular. Wi-Fi would also work, but it would limit where the sensor could be placed.

Adafruit designs and manufacturers electronics that are perfect for prototyping. Their products are designed to be accessible and are well documented. They have standardized a form factor for development boards known as Feather. The Feather ecosystem is large and offers a wide range of functions. There are Feather boards available the offer MicroSD storage and also ones with LoRa radios. An expansion board, like a Feather Tripler, lets you connect multiple feather boards together.

![3boards](./media/3_boards.jpg)

The Arduino Nano 33 BLE board is based on the Nordic RF 52840 processor. Luckily, Adafruit has recently released a nRF52840 based board, in the Feather form factor. It has a lot of onboard sensors, including a microphone. The board also includes a battery connection and charging circuit. Adafruit previously released other Nordic RF based boards and has developed software that adds [Arduino support](https://github.com/adafruit/Adafruit_nRF52_Arduino). Using the following Adafruit boards, I was able to meet my design requirements:
-	[Featherwing tripler](https://www.adafruit.com/product/3417) This lets you easily connect together multiple Feathers
-	[Adafruit Feather nRF52840 Sense](https://www.adafruit.com/product/4516) This development board combines it all. It has a Nordic RF 52840 processor which is great for ML, with 256kb of RAM, 1MB of storage and is Cortex M4 based. There is also an onboard MEMS microphone. It can be power using LiPo batteries or over USB.
-	[LoRa Radio FeatherWing](https://www.adafruit.com/product/3231) This board adds a LoRa radio. You can select different pins to use for communicating with the LoRa chip, based on how you solder jumper wires. Having to solder the jumper wires is a bit of a pain, but it allows you to avoid conflicting with pins used by other boards. You can choice between using a u.FL or SMA connection for attaching an antenna.
-	[AdaLogger FeatherWing](https://www.adafruit.com/product/2922) This board gives you are battery backed real-time clock and a way to save data to a microSD card.
-	[6600mAh LiPo Battery](https://www.adafruit.com/product/353) This battery has a lot of power but is not super large. It can provide multiple weeks of power for the sensor.

While it would have been possible to design a 3D printed case, I  decided to go with one of the many weatherproof cases that are available. [Polycase](https://www.polycase.com) has a lot of great choices. Since I was going to be opening it frequently to update software and collect recordings, I selected [one](https://www.polycase.com/wh-02) that is closed with latches instead of screws.

![polycase sensor box](./media/sensor_box.jpg)

### Hardware Gotchas
Whenever you build something, there are always unexpected surprise and building this audio sensor was no exception. Overall, each of the individual components worked well on there own. However, things got interesting when I started combining them together. Here are some of the things I encountered:

#### Microphone 
When you put a microphone in an enclosure, interesting things happen. First off, you need to make a hole to help let sound waves in. That is not enough though. You also need to correctly position the microphone near the hole. If the microphone isn’t directly against the hole, some frequency can start to become muffled, while other are amplified. Essentially, the project enclosure acts like a speaker cabinet. There is a lot of [literature](https://www.infineon.com/dgdl/Infineon-AN557_MEMS_microphone_mechanical_and_acoustical_implementation-AN-v01_01-EN.pdf?fileId=5546d4626102d35a01612d1e3bf96add) on how to correctly position a microphone in a housing. 
In short though, your life will be easier the closer microphone is to the hole and the larger the hole is. 

I learned about this issue through experience. I initially drilled a hole in the front of the case and then positioned the board, aligned with the hole, in the back of the case. All of the empty space between the hole and the mic distorted the sound. I noticed this because the accuracy of the ML model was off. I only figured out that the actual sound being captured was the problem because I happened to be recording some of the collected samples. This highlighted the value of being able to review the raw samples being collected.

To correct things, I attached the boards to the lid of the container, allowing for the microphone to be positioned against the hole. The boards have a number of components on top of them, making it impossible for the microphone to be laid flat against the lid. To help better connect the mic to the hole, I cut a thin piece of foam and fitted it around the hole. I also added some thicker pieces of foam to help fill in the empty space in the enclosure. Combined, both of these things seemed to do the job.

#### Antenna Connection
![ufl](./media/ufl.jpg)

 The LoRa board allows you to either use an uFL or SMA connection to attach an antenna. I originally used an uFL connection so I could connect the board to an SMA connector outside the case. Having an external SMA connector made it easy to attach a large antenna. While this worked marvelously at first, the uFL connection is not that mechanically strong. After having to move the board around a couple times, the mechanical strain must have been too much and part of the connector popped off. After that, I switched over to an edge mounted SMA connector. It is more securely attached to the board because it attaches on both the top and bottom. The connection between the cable/antenna and connector is also a lot stronger and is screwed on instead of being press fit. Right now, I am just using a small antenna that is able to fit inside the enclosure. I think it would be possible to use a small extension cable to allow for an external SMA port, or just use an externally mounted  antenna with a long SMA cable, feed through a cable gland. My big lesson learned is not to use uFL connections if things are going to be moving around a lot. (They are also a pain to solder on.)

#### Pinouts 
![lora](./media/lora.jpg)

While the Feather is a fabulous ecosystem, you do have to pay attention to the which boards use which pins as you start combining them. This bit me when I tried to use the both the LoRa Featherwing and the Aadalogger. The Adalogger has a fixed set of pins it uses, while the LoRa board lets you configure what pins to use, by soldering some jumper wires. I, of course, just soldered the jumpers in a 1,2,3 configuration without checking. This unfortunately caused a pin “collision” and caused me a bit of confusion until I figured it out. I did get to practice my desoldering skill while correcting the jumpers. Lesson learned here, double check your pinouts before you solder!

The correct wiring is:
- C -> RST
- D -> CS
- E -> IRQ
 
Board Pinouts:
- LoRa Featherwing [Pinouts](https://learn.adafruit.com/radio-featherwing/pinouts)
- AdaLogger Featherwing [Pinouts](https://learn.adafruit.com/adafruit-adalogger-featherwing/pinouts)


## Software for the Sensor

The software running on the sensor started from the example audio classification program provided by EI. To further improve the model, I needed to be able to evaluate how well it worked in the real world. I also wanted to also collect audio samples that could be labeled and used as training data later. Recall that the model was originally built with audio from the internet. To do both of these things, I needed to be able to record the audio captured by the dev board. Unfortunately, there are not a lot of examples on how to record audio to a file in Arduino and also how to listen to the file on a computer. Capturing digital audio is rather processor intensive. The signal needs to be sampled 10s of thousand times per second, and then has to be transferred into memory that the processor can access. On the nRF52840, this is luckily handled by a [subsystem](https://infocenter.nordicsemi.com/index.jsp?topic=%2Fcom.nordic.infocenter.sdk5.v15.0.0%2Fhardware_driver_pdm.html). All you need to do, is tell the processor the sampling mode you want to use, the gain to apply and then an interrupt to contact us on when audio samples are ready. 

The processor supports a few sample rates, but the Arduino core only [supports](https://github.com/adafruit/Adafruit_nRF52_Arduino/blob/4d703b6f38262775863a16a603c12aa43d249f04/libraries/PDM/src/PDM.cpp#L66) 16khz or 41.667khz. Luckily the model was trained with 16khz audio. 

When the processor has filled its buffer with captured audio, it fires an interrupt, which notifies the Arduino that it needs to move the samples from the buffer into memory the program is using. The Adafruit has a digital MEMS microphone that sends the audio to the processor using PDM. The processor converts the PDM signal into PCM data which is much easier to work with.

There are some examples of the different formats here:
https://devzone.nordicsemi.com/nordic/nordic-blog/b/blog/posts/pdm-example-on-the-nrf52832

A 16-bit resolution is used for the samples, stored in 2 sequential bytes. Of course, when you start using more than one byte to store a value, you have to start worrying about processing and storing them in the correct order. This is known as [endianness](https://en.wikipedia.org/wiki/Endianness).

If you mix the order up, you'll notice pretty quickly. Each sample is a represented as a number and the larger the number is, the louder it is. To make something twice as loud, you multiply each sample by 2. I noticed I had screwed the endian up because the audio I was saving was really quiet, with occasional burst of distortion. 

The most straightforward way to run an ML model against captured audio, is to copy samples from the processor each time the interrupt is called and accumulate them in a buffer. Once you have a complete sample, in my case 2 seconds worth of audio, you run it through the preprocessor, convert it into features and then pass it through the model. Since you have a complete copy of the audio sample in memory, you can also write it the SD card, along with the prediction from the model. This will let you listen to the audio later and evaluate how well the model did. The samples are simply an array of PCM data, only a small amount of manipulation is needed to transform it into a .wav file that can be played back on a computer.

While this approach is great, it means that you are flip flopping between recording audio and running the model. While the model is being run, nothing is being captured, allowing you to potentially miss events. If there is enough memory available, you can use 2 buffers, allowing for one to be written to while a model is being run on the other. If your model and sample size are small enough, you can get away with using this double buffering. Assuming the times it takes to process your audio and run a model against it is less that your sample size, this approach will let you continually detect events.

There is an additional approach that I tried out, which allows for using a single buffer but still provides continual detection. Instead of waiting for a sample to be completely captured before running a model on it, you do being running the model on the audio as it comes in. This operates like a circular buffer. There are 2 pointers into the buffer. One for where the next bit of audio should be written to and another for where the next audio sample should be read from. When a pointer reaches the end of a buffer, it will wrap around to the beginning. For this to work, you need to make sure the reading pointer doesn’t go past the writing pointer. The audio processing and running the model also have to be faster than the rate at which new audio is coming in. If it can’t keep up there will be gaps in the audio being feed in because samples are being written over. The whole scheme is a bit complex, and I am sure there are some subtle bugs that are lurking in my implementation. The other drawback is that it would be tough to save audio samples and model predictions for them to an SD card.  However, this could be a useful technique when you are just trying to run a model continuously, but have memory constraints.

### Model Deployment

![deployment](./media/deployment.png)

After you have finished training a model in Edge Impulse, you can goto deployment and export it as an Arduino library. In the Arduino IDE, goto the Sketch Menu, the Include Library and then Add .ZIP Library. Find where you downloaded the .zip file with you EI model and import it. This will add it the **Libraries** folder, inside your **Arduino** folder. 

The EI library for running your model can make use of CMSIS_NN for both inference and the DSP work. Unfortunately, it doesn't automatically detect that the Adafruit board is capable of CMSIS_NN. With a few tweaks though, you can lock it in. Make the following changes to the library you add into the **Arduino/Libraries** folder.

***ei-project*/src/edge-impulse-sdk/classifier/ei_classifier_config.h**

```c
#elif defined(__TARGET_CPU_CORTEX_M0) || defined(__TARGET_CPU_CORTEX_M0PLUS) || defined(__TARGET_CPU_CORTEX_M3) || defined(__TARGET_CPU_CORTEX_M4) || defined(__TARGET_CPU_CORTEX_M7)
    #define EI_CLASSIFIER_TFLITE_ENABLE_CMSIS_NN      1
#else
    #define EI_CLASSIFIER_TFLITE_ENABLE_CMSIS_NN      1
#endif
```

***ei-project*/src/edge-impulse-sdk/dsp/config.hpp**

```c
#else
    #define EIDSP_USE_CMSIS_DSP		        1
    #define EIDSP_LOAD_CMSIS_DSP_SOURCES    1
#endif // Mbed / ARM Core check
```


## Dataset Expansion

With this sensor I was now able to better understand how the model worked in the real world and expand the dataset to improve the model’s performance. The sensor recorded the 2 second samples of audio it captured to a microSD card, along with a prediction from the model and its confidence. Having the prediction from the model allowed me to focus on the samples where the model was uncertain, apply labels and add them to the dataset. In reviewing the recordings, I found that there were not that many sirens detected. I reviewed all of the detections and found that the model did a remarkable job of correctly detecting sirens, even when they were distant. I did find a few false positives though. The model often mistook the warning beeps a truck makes while backing up with a siren. I corrected the labeling on those samples, but I was never able to fully prevent these false positives. I suspect that after feature extraction, backup beeps and sirens may look identical. It is possible that switching to MFE or Spectrogram processing may help the model differentiate the two. 

The majority of the recorded samples where predicted to be noise. At this stage of development, I didn’t want to listen to ~23 hours of silence because I was expecting some prediction mistakes. Instead I did spot checks of different samples and found that the model was doing a great job. I did not find any instance where the model label a siren as being noise with high confidence. 

There were a few instance where the model was very uncertain, having around 50% confidence. After reviewing those samples, the model may have been right to be confused. After all, what is even a siren? Emergency vehicles make a lot of different noises to get your attention. Some of these are siren sounds... others are more like a horn. I came to realize that I should have come up with a more formal definition of siren to use when I was labeling samples. Not all sounds that an emergency vehicle makes should be considered to be sirens.

I did a number of iterations on this process, running the model, collecting samples, correcting labels, and retraining. From doing I came away with a more accurate model and some observations. Sirens are a continual sound that will last much longer than the 2 second sample. While it is insightful to look at the accuracy of predicting a single sample, operationally the sensor should look for multiple consecutive samples where is a siren is detected. When this filtering of the predictions was used, it appeared that the sensor was working flawlessly. It did not appear the miss any siren events or have any false detections. Time to formally verify this!

## Sample resolution: 8KHz vs 16KHz

There is a direct relationship between the amount of compute and memory you need, and the sample rate of your data. Audio recorded at 8KHz means that there 8,000 samples being produced every second, 16,000 samples with 16KHz. 16KHz audio seems to be a good default for getting started. Useful sized samples can fit into memory and there is not too latency in processing them. While using 2 second samples at 16KHz, I did not have enough memory to allocate 2 buffers in order to continuously process the audio. To allow for this, I decided to investigate using 8KHz audio.

Using common tools, it is pretty easy to downsample audio to a lower sample rate. On a Mac, the following command will downsample all the audio in a directory to 8KHz.

```bash
for i in *.wav; do ffmpeg  -y  -i "$i"  -ac 1 -ar 8000 "8khz/${i%.*}.wav"; done
```

After doing this, I then created a new project on EI and uploaded the resampled audio. With the same signal processing and neural network configurations, performance was surprisingly similar:

**8KHz Performance**
![8khz performance](media/8k-performance.png) 

**16KHz Performance**
![16khz performance](media/16k-performance.png) 


## A More Rigorous Evaluation

While I had been doing some rough confirmation of performance using the approach I outlined above, to really evaluate the system I needed a more formal methodology. In order to capture a range of environmental noises, I let the sensor run for 24 hours. I then went and listened to 24 hours of recordings, broken up into 2 second clips, and added ground truth data to compare the predictions to. This was not fun. Luckily, using VLC, I was able to speed up the playback. 

The EI Arduino library supports performing inference on a single sample or [continually](https://docs.edgeimpulse.com/docs/continuous-audio-sampling) performing inference over a sliding window and returning a moving average. In order to ensure the inference scores directly lined up with the clip that got recorded to the MicroSD card, I used single sample inference.

After all that listening, I finally had some ground truth data. With this, I could generate a confusion matrix.

![Evaluation Confusion Matrix](media/eval-confusion.png) 


At first glance, it looked like a bit of a mixed bag. The good news was that there were very few false positives. That less good news was that when there were sirens, it only recognized them about half the time. A little dismayed, I decided to dig into the numbers and it turns out that the story was a little more complex that it initially appeared. For the evaluation, I had set a confidence threshold of greater than 80% for declaring something a siren. If it is adjusted to be great or equal to 75, you would detect an additional 3 sirens, while only adding 2 additional false positives. 

It is more interesting to look at things as being events, instead of single instances. This lets us see how we are doing at detecting when an emergency vehicle passes by.

----

### Siren Event 1
 
Prediction |	Timestamp |	Confidence | Ground Truth
:--------:|-------------|:----------:|:-----------:	 
siren |	14:19:04 |	82 |	TRUE
**noise** |	14:19:08 |	**77** |	TRUE
**noise** |	14:19:11 |	**1** |	TRUE
**noise** |	14:19:36 |	**0** |	TRUE
siren |	14:19:39 |	82 |	TRUE
siren |	14:19:43 |	99 |	TRUE
siren |	14:19:46 |	99	| TRUE
**noise** |	14:19:50 |	**75**	| TRUE
siren |	14:19:53 |	99	| TRUE
siren |	14:19:57 |	99	| TRUE
siren | 14:20:00 |	99	| TRUE
**noise** |	14:20:04 |	**13**	| TRUE
**noise** |	14:20:07 |	**80**	| TRUE
siren |	14:20:11 |	99	| TRUE
siren |	14:20:14 |	98	| TRUE
siren |	14:20:18 |	92	| TRUE
siren |	14:20:21 |	98	| TRUE
**noise** |	14:20:25 |	**24**	| TRUE
**noise** |	14:20:28 |	**2**	| TRUE
**noise** |	14:20:32 |	**0**	| TRUE


### Siren Event 2
 
Prediction |	Timestamp |	Confidence | Ground Truth
:--------:|-------------|:----------:|:-----------:	 	 	 	 
siren |	23:13:15 |	98	| TRUE
siren |	23:13:18 |	99	| TRUE
**noise** |	23:13:21 |	**55**	| TRUE
**noise** |	23:13:27 |	**17**	| TRUE
siren |	23:13:30 |	99	| TRUE
siren |	23:13:33 |	99	| TRUE
**noise** |	23:13:36 |	**0**	| TRUE

### Siren Event 3
 
Prediction |	Timestamp |	Confidence | Ground Truth
:--------:|-------------|:----------:|:-----------:	 	 	 	 
siren |	6:58:48 |	85 |	TRUE

### Siren Event 4
 
Prediction |	Timestamp |	Confidence | Ground Truth
:--------:|-------------|:----------:|:-----------:	 	 	 
**noise** |	9:36:25 |	**1**	| TRUE
**noise** |	9:36:29 |	**0**	| TRUE
**noise** |	9:36:33 |	**0**	| TRUE
**noise** |	9:36:37 |	**1**	| TRUE
**noise** |	9:36:41 |	**0**	| TRUE
**noise** |	9:36:45 |	**3**	| TRUE
siren |	9:36:49 |	82	 | TRUE
siren |	9:36:53 |	98	 | TRUE
siren |	9:36:57 |	89	 | TRUE
**noise** |	9:37:01 |	**8**	| TRUE
siren |	9:37:05 |	95	 | TRUE
siren |	9:37:09 |	92	 | TRUE
siren |	9:37:13 |	99	 | TRUE
**noise** |	9:37:17 |	**0**	| TRUE
**noise** |	9:37:21 |	**2**	| TRUE
**noise** |	9:37:25 |	**1**	| TRUE


Things look quite a bit better. The detector didn’t miss a single event. One possible reason for the spotty performance is because emergency vehicles have a range of sounds they can make and the model maybe overtraining on one type. This goes back to better defining what constitutes a “siren”. 

When we look a timeline of the false positives, where a siren was detected, but their actually wasn’t one, things look a bit different:

---
### False Positives
Prediction |	Timestamp |	Confidence | Ground Truth | Source
:--------:|-------------|:----------:|:-----------:|:-----------:	 
siren	| 15:03:41	| 91	| FALSE |	Nothing
siren	| 16:18:25	| 99	| FALSE |	Break Squeal
siren	| 6:28:07	| 85	| FALSE	| Nothing
siren	| 11:32:35	| 91	| FALSE	| Backup plus leaf blower
siren	| 11:48:41	| 85	| FALSE	| Unknown
---


Unlike the true positives, none of the false positives were consecutive. The most common source was the backup beeping that trucks make. It is possible that additional data samples and retraining may have solved this false positive. It is also possible that the feature extraction being used, ends up making backup beeps and sirens looking the same. Looking at the features that get generated may help diagnose this. A one dimensional convolutional neural network architecture was used for the model. Switching to a 2 dimensional network may help it distinguish the 2 better.

After comparing true positives and false positives, it is clear that filtering the results can help improve accuracy. Simply filtering for 2 consecutive siren detections would remove all of the False Positives. It would still allow for the detection of all of the siren events, minus that weird one where there was only a single siren sample. 

One clear takeaway is that if you are trying to detect continuous sound and you can afford the latency of waiting a couple samples, you should be doing some filtering. 

## Next Steps

My big take away is that the model does a good job of detecting sirens. It is unlikely that there will be a siren and the model will completely fail to detect it at all. This is great news!! It means that for future experiments I don't need to listen to all the audio... yeah! 🙌

For future experiments I want to try running 2 identical setups side by side, one running a model with 8KHz sampling and one with 16 KHz sampling. 

I also did all this work in the Fall of '20... and it just took me forever to write it up. Since I did it, EI released a Spectrogram Feature Generator. I am curious to see the performance difference compared with MFCC.

I also want to implement continuous averaging against the inference results. The easiest way would be using the built-in EI function: [run_classifier_continuous()](https://github.com/edgeimpulse/inferencing-sdk-cpp/blob/master/classifier/ei_run_classifier.h#L197) and using the built in smoothing functions to filter: [ei_classifier_smooth_init() & ei_classifier_smooth_update()](https://github.com/edgeimpulse/inferencing-sdk-cpp/blob/master/classifier/ei_classifier_smooth.h#L49). There are examples of how to use them in the Accelerometer Continuous Arduino Example. 

If I add in this filtering, I would feel pretty confident in the result coming from the sensor... and I can finally start to answer the question of, How many times a day do you actually hear a Siren?

Finally, I also want to try and get the dataset I collected out there. I am pretty sure the crowd can make further improvements!

## Conclusion
---
After doing all this work, my big takeaway is that I think there is something useful here. This technology is accessible and a lot more people are going to be creating ML powered sensors. Tools like Edge Impulse have made it easier to integrate ML into embedded devices. Adafruit's Feather ecosystem provides great modularity, making it easy to create a custom system. There will be certain scenarios that TinyML is better suited for than others, but this will improve over time as the hardware gets more powerful and new techniques are developed. 

### TinyML is exciting!
