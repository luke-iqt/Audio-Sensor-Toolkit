# Overview

![boards](./media/artsy.jpg)

I have been working with embedded systems for a rather long  time. I am by no means an expert, but I have managed to keep up with the different embedded platforms as they have emerged. However, I am very much a beginner  when it comes to Machine Learning (ML) and I suspect there are a lot of other folks on the embedded side that are too. Adding ML to embedded devices has a lot of potential, and has mature to the point where it can start being useful and not just a novelty. This guide will walk through a project to build a wireless sensor that can detect sirens using ML. The core of what is presented here can be extended to easily detect other types of sounds.

## Project Goals
Working from home can lead to some realizations. For me, one of those was, “Man, there sure are a lot of sirens”. This happened to coincide with my current fascination with adding ML to sensors, and from this, a project was born. The formal project goal was to build a sensor that can accurately count how times sirens could be heard during a day, and see how that day compares to other days in the week.

While the geneses of the project was rather whimsical, there are practical applications. For the most part, whenever you hear a siren it is because something happened. If you hear a siren for a long time, at a loud volume, the event is probably happening nearby. Integrating siren detection into a security camera could give you a pulse of the neighborhood. 

## Related work
There is also a lot of interest in identifying the sources of urban sounds. Excessive noise can have a real impact on quality of life. The [Sound of New York City](https://wp.nyu.edu/sonyc/) project is a research effort to catalog and map the sources of urban noise, including sirens. This effort has been mostly focused on post processing recordings to measure the different types of sounds. This is understandable because it has only recently become possible to do this completely on-device. [Jon Nordby](https://www.jonnor.com/tag/environmental-sound-classification/) examined doing on-device classification of 10 common sound class using an embedded device in his [master thesis](https://github.com/jonnor/ESC-CNN-microcontroller). He found that it was possible to run an optimized model within the processing and memory constraints of an embedded device. While there was some reduction in accuracy, it would work well enough to provide an understanding of the largest contributors to overall noise throughout a span of time. Testing for this work was performed in the lab. As we discovered, real world testing is a critical element of evaluating an approach.

### [Next Chapter: Design Considerations](./design.md)