# Pytorch-Research-of-Image-Deblurring-Based-on-the-Deep-Neural-Network
This repository is Clean and scratch implementation of research paper **Research of Image Deblurring Based on the Deep Neural Network**. Famous Celeba Face [Kaggle dataset](https://www.kaggle.com/jessicali9530/celeba-dataset) was used in this experiment. [Link to paper](https://ieeexplore.ieee.org/abstract/document/8405801/)! This project was being carried out as a semester project of **CS5102-Deep Learning** at [**NUCES ISB**](http://isb.nu.edu.pk/). Including me, three group members carried out this project. [Training Notebook](https://drive.google.com/file/d/1iWkjXSpLcAhqaONZrOCX9uK8z0vSQiDD/view?usp=sharing)!

## Dataset Preparations
Famous Celeba Face dataset was downloaded from Kaggle. It has a total of 290K+ images. Entire publically available dataset was being used for this experiment. We blured dataset with a mixture of *Guassian* and *Motion* blur. Research Paper didn't specified any particular blur, so we manually perfromed motion blur on half 148K images and performed guassian blur on other 140K+ images.

Now we had two datasets saved in two different directories with identical names of images. e.g. 1.png in original directory, and corresponding blured image 1.png in blur directory. Our `getitem` method in custom `Dataset` class would load both paired images from original and blur directories. After performing below transformations on both images, method will return original image as label of blur image in a list like this `return [org_transformed_img, blur_transfomred_img]`.

* `transforms.ToTensor()`
* `transforms.Resize([image_resize, image_resize])`
* `transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])`

## Network Architecture
We used skip connections int convolution layers. There was someone sync error in flow of network in original paper. So needed to change parameters and kernels size and number to make network go forward and backward without shape mismatch errors. Below image is taken from paper. ![Network Image taken from Paper](/network.png)
### Generator - Fully Convolutional Auto Encoder Decoder with skip Connections
Proposed generaotr was a Fully Convolutional Auto encoder network containing skip connections at multiple level. ![Generator Image taken from Paper](/generator.png). For Decoder we are Using torch.nn.ConvTranspose2d to scale up. We needed to change it as according to given architecture lateral size and kernel sizes, flow of network was errorneous. Our modified network was as follow:


Input-Channel | Output-Channel | Kernel-Size | Stride | Output-Lateral-Size | Skip-Connection-Layer
----------------|----------------|-------------|--------|---------------------|------------------------
| | | **ENCODER** | |
3 | 64 | 3 | 1 | Output 62x62 | E1
64 | 128 | 3 | 2 | Output 30x30 | E2
128| 256| 3| 1 | Output 28x28 |E3
256| 512| 3| 2 |Output 13x13 | E4
512 | 512 | 3| 1 |Output 11x11 | -
| | | **DECODER** | |
512| 128| 3| 1 |Output 13x13 | Concatenated with E4
640| 128| 4| 2 |Output 28x28 | Concatenated with E3
384| 64| 3| 1 |Output 30x30 | Concatenated with E2
192| 32| 3| 2 |Output 61x61 | -
32| 16| 2| 1 |Output 62x62 | Concatenated with E1
80| 3| 3| 1 |Output 64x64 | -
### Discriminator

Our discriminator is a binary classifier, responsible for detecting fake images generated by the generator. Our model follows a Batch Normalization layer after every **Convolutional layer**. We used Relu Activation function between all of our layers. Discriminator network ends with a *Sigmoid* function. The construction blocks for the discriminator are as follows.

|Input-Channel | Output-Channel | Kernel-Size | Stride | Activation |
|--------------|----------------|-------------|--------|------------|
|       3      |       64       |      3      |   1    | ReLU      |
|64 |  |  |  |  **BatchNorm2d** |
|       64     |       128      |      3      |   2    |ReLU      |
|128 |  |  |  | **BatchNorm2d** |
|       128    |       256      |      3      |   1    | ReLU      |
|256 |  |  |  | **BatchNorm2d** |
|       256    |       512      |      3      |   2    | ReLU       |
|   512x13x13    |       1 | **Linear** |  | Sigmoid |



## Loss functions
 **Descriminator :** For the descriminator(D) side, the paper is using the standard adversarial loss, described as follows.

![Adversarial Loss](/d_loss.png)

 The descriminator(D) tries to maximize the above function and the generator(G) tries to minimize it. In other words we want our descriminator(D) to correctly classify "Original Images" as valid while at the same correctly classifying "Generated Images" as invalid, therefore giving us this adverse relationship between the two networks.

<br>
 
 **Generator :** In standard GAN models, a slightly modified BCELoss is typically used in-order to train the generator part. It is slightly modified because instead of passing in the original data to **D** we are giving it the output of **G**.
  ![BCELoss](/bce_loss.png) 
 
  In this paper's implemenration, we are using a joint loss function consisting of following two parts.

  * [SSIM-Loss](https://github.com/Po-Hsun-Su/pytorch-ssim) (Structural Similarity Loss).

  * [BCELoss](https://pytorch.org/docs/stable/generated/torch.nn.BCELoss.html).

  These two losses are added together into a joint loss and the gradients are then computed based upon this joint loss.

  > SSIM loss is being used to preserve structural similarity of images.

  > Also inorder to prevent our descriminator getting more powerful than our generator, for every descriminator training iteration the generator is trained 3 times.

  ## Results