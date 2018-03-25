# Feature Pyramid Networks For Object Detection

This paper aims to provide a fast way to construct feature pyramids for different scales of an image. Previous pyramid representations took up a lot of time/space to compute
Here a Feature Pyramid Network is constructed which takes into account the intrinsic nature of conv nets to construct feature pyramids at marginal cost.


# Author 
Facebook AI (FAIR) 

## Paper Link
https://arxiv.org/pdf/1612.03144.pdf

# Nomenclature

```
@ -> Matrix Multiplication
* -> Hammard's Product
```

## Introduction

1. This model unlike previous models , creates a feature pyramid that combines ```low resolution, semantically strong features with
high resolution, semantically weak features``` using ```top down pathway and lateral connections.```

2. Previous Models like SSD , used last layers to compute feature pyramids hence skipping high resolution maps, thereby decreasing accuracy in detecting small objects

3. Aim is to create an in network feature pyramid, hence saving on time/space

4. The authors incorporated a FPN in Faster R-CNN and achieved state of the art results

5. Hence network can be trained at all scales and is used consistently at test/train time.

# Model

1. Resnet used to construct feature maps.

2. There are two things, bottom up and top level pathways
    * Bottom Up Pathways are the *feature maps* after each residual block (in the Resnet) , hence denote features at different depths
    * Top Level Pathways are upsampled features from higher level, smaller feature maps. Upsampling is done with a factor of two using (nearest neighbour upsampling)

## Combining these layers

1. The two, top level and bottom up layers, of the same size are combined using addition. Hence is the cae of resnet you will have a final 4 layers. Conv1 is not used as it takes up a lot of memory.

2. Bottom up layers are undergone a 1 x 1 conv to bring them to the same channel dimensions.

3. The final layers are made to undergo a further 3 x 3 conv . Hence getting P2,P3,P4,P5.

4. One channel dimension is used ,that is = 256.

# Modifications to existing Networks to Incorporate FPN's

## FPN for RPN (Region Proposal Network)

### RPN

The RPN Network is a network first described in the Faster RCNN paper. It was meant to seperate the foreground and background anchors. 

Anchors are basically meant to denote , image regions at a particular pixel point as its center, and anchors can have a variety of aspect ratios. (1:1,1:2,2:1) and scales (32,64,128) etc. The faster RCNN paper has 9 scales. For 3 scales and 3 resolutions.
```
Anchors are fixed bounding boxes that are placed throughout the image with different sizes and ratios that are going to be used for reference when first predicting object locations.
```
The RPN takes in an image feature map , outputted from a convolutional base. And attaches two heads to each anchor point. The two heads are:

```
1. The Classification head. A softmax layer that has 2 classes applied over all anchors.
2. A regression head, that outputs 4 values for each anchor. delta _delta_x,delta_y,delta_h,delta_w_  All denoting errors in x, y (center coordinates) and height and width. To fix/modify the static anchor shape.
```

Anyway , more details on the Faster RCNN note. Please check that. 

All we need to know is that an RPN gets us a plausible set of anchors. Decided by taking the ones have an IOU of more than ~0.5 from the ground truth boxes.

Now, onto the interesting part.

### FPN in RPN

Unlike the RPN used in Faster RCNN, this FPN-RPN does not have different scale anchors and does not use one feature map. 
It uses multiple feature maps, from the feature pyramid of the FPN and uses one scale per feature map foir the anchors.

The intuition is that the feature pyramid deals with the multiple scales. 
```
Hence the anchors are said to have areas of 32^2, 64^2, 128^2, 256^2, 512^2 for P2, P3 ,P4, P5 ,P6 (pyramid levels)
```

The anchor ratio remains the same though . Hence 3 x 5 = 15 anchors per pixel point in the image.

The main difference in the RPN with an FPN is that *the multiple scale anchors are handled by the different feature pyramids in the FPN*. 


The labels are generated by associating anchors with the ground truth boxes. The box coordinates for an anchor is calculated, it's IOU (Intersection Over Union) is calculated with all the ground truth boxes. _How the box coordinates are exactly calculated is not known as of now, will update as soon as I get to know it._ 

```
I am assuming , the pixel is tracked back, say a 32^2 area is calculated by tracking back from P^2 . Then after applying the correct aspect ratio, the final label box is generated and IOU is calculated
```

Coming back on topic,if the IOU is the largest of any other anchor or it is more than 0.5, the anchor is classified as foreground, else if IOU is less than 0.1 , anchor is classified as background.

During training , like the vanilla RPN. 
```
256 anchors are chosen , balanced within foreground and background labels.
```
And loss is calculated for them and the gradients are updated. More than 256 anchors are not chosen as I think , the authors thought that as most of the anchors would be background, the training set will be severly imbalanced. Slow training time will be substantially increased as the number of anchors would go up to ~100k.

```
Feature maps are shared by the FPN-RPN throughout all pyramid levels.
```
## FPN for R-CNN
