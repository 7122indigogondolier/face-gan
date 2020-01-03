# Implementing Deep Convolutional Generative Adversarial Learning of Realistic Human Faces 

Implemented by : Griffin Noe and Utkrist P. Thapa

Washington and Lee University



![Final Result](https://raw.githubusercontent.com/7122indigogondolier/face-gan/master/final.png)

This is a Python implementation of a DCGAN (Deep Convolutional Generative Adversarial Network) that generates faces in (128,128,3). We use CelebA dataset found here: http://mmlab.ie.cuhk.edu.hk/projects/CelebA.html. This implementation is based on Tensorflow and Keras. 



## Getting the Data
There is not much frontloaded data work as we do most of the data operations when we retrieve individual images due to the high number of images. 

We first had to download the dataset at the above link and unzip the 200,000+ images into a local directory. Because there is so much data, we only access single images at a time and do preprocesses/deprocessing when we evoke the image instead of running operations on the entire dataset at the beginning. This obviously decreases the time it takes at the forefront to run the program which is important because the model saves at multiple points. This means that the faster it can get to the first 500 batch iterations, the faster we can start to evaluate the quality of the generated images. 

We use the glob module to extract all the filenames in the directory of images and store them in a numpy array named filenames. We then use the train_test_split function from scikit learn to split our data with 1000 into x_test and the rest (201,599) into x_train. Again it should be noted that this operation only occurs with the numpy array of filenames, not the actual images. While a training/testing split of 201,599/1,000 seems like an absurd choice, for the purpose of GANs the testing set is not extremely important. The training set is where we actually tune the ability of the generator to deceive the discriminator and this tuning is the point of the entire GAN in this case. Therefore it is much better to have an extremely effective generator that has an overtrained discriminator than to have a mediocre generator with a perfectly trained discriminator. This is why we split the training/testing set to 201,599/1,000. 



## Data Helper Functions
We created a set of helper functions that were used for various operations on the images and their pixel values, sizes, shapes, etc. 

### load_image:

load image is the function that was previously mentioned that loads individual images from the directory when given an image size and the filename of that given image. This function both crops and resizes the image as the original image size of (218,178,3) was a very large image and also not an easy size to work with for a clean CNN. We first read the image in from the file name using pyplot's imread function. We then extract the width and height (r and c for rows and columns) of the image as well as the crop size we want to cut the image to. We chose 150 pretty arbitrarily as we saw many other networks use this number and we found that the images cropped to 150x150 still displayed the entirety of the celebrity's face in almost every case. We then run some basic algebra operations based on the size of the images and the size of our cropping area to set image to its newly cropped self. We then resize the cropped image to the image size specified in the function call and return that cropped, resized image. These operations are all extremely important for both minimizing training time of the network and for the ability of the generator and discriminator to be generally clean models with them getting fed in, and spitting out, clean square images. 

### preprocess:

This preprocess function is used because our generator uses the tanh activation function, which needs input data in the range of (-1,1) so we first divide the pixel value by 255 so it is between 0 and 1 then we multiply by two and subtract by one which ensures a pixel value in our tanh range. This pixel normalization is extremely important because the activation function would simply be unable to handle any images if the pixel values were not in its range. 

### deprocess:

The deprocess function simply does the inverse of preprocess and reverts the pixel values to their unsquashed values so we can display the images later. This is obviously important because we could train the model with pixel values entirely in the tanh range but would never be able to see the images unless we put the pixel values back into their natural range. 

### show_losses:

This function takes the losses arrays created in our training method and plots them. This is an important visualization in evaluating the accuracy of our generator, discriminator, and their relative accuracy. These plots are displayed in our paper for different architectures and training iterations. 

### show_images:

This function takes generated images from our generator and displays them in a plot of 80 randomly selected images. This is another important visualization as the generator and discriminator losses don't always accurately convey the message and sometimes a subjective evaluation of the images created by the generator is a better marker for success than just validation scores and losses. 



## Model Functions
As this is a generative adversarial network, we had to construct a generator, a discriminator, and then put them together in order to create our GAN. 

### create_generator:

This is the function we use to create our generator network. The summary for this architecture can be found in our paper. 

We take the number of initial filters for the convolutional layers, 
the input size is the size of the noise vectors fed into the generator, 
alpha is the alpha for our leaky ReLU layers, 
stdev is the standard deviation used for our kernel initializer within the convolutional layers, 
kernel_size is the size of our kernels in the convolutional layers, 
strides is our stride size for the convolutional layers,
padding is the type of padding we use in our convolutional layers, 
and momentum is the momentum we use in our BatchNormalization layers. 

This generator is instantiated using the Sequential Keras model. We start by adding a dense layer that reads in the noise vector followed by a reshape layer that acts as a connection between the 100 dimensional latent vector and our first transposed convolutional layer. These layers are obviously necessary to the ability of this network to create images because it is fed in a 100 dimensional noise vector that it needs to convert into a 64,64,3 image and these two layers are the first step. They basically serve as a way to transform the randomized noise vector into a general image shape so that the convolutional layer is able to handle the input. We needed a dense layer because all of the information from the noise vector should be passed through into the first convolutional transpose layer. The kernel initializer is used in nearly every real layer (excluding activation functions, batch normalizations, etc.) and we decided upon a randomized normal distribution with a very small specified standard deviation of 0.02. At first we did not specify a kernel initialization and instead used the default (glorot_uniform) but found that this distribution was too wide and the randomization of the noise vectors had a large outcome on the performance of the model which greatly decreased accuracy. Finally within this first section of our generator, we add a batch normalization layer. When we first created this model, we did not have a batch normalizer between every layer. We found that the network would only run semi-successfully with extremely small learning rates (about 1/10 of what we currently have) and even then, training took an extremely long time (at the time we had a generator in (32,32,3) and a discriminator in (64,64,3) and each batch iteration took about 15 seconds compared to our final architecture of a generator in (128,128,3) and a discriminator in (128,128,3) that has a batch iteration time of around 6.5 seconds). We researched ways to improve the training time and found that implementing batch normalization layers would decrease network training time by allowing for significantly higher learning rates while also increased accuracy in the models. We decided to tweak the momentum of the batch normalization layers down from their default of 0.99 to 0.9 so that we would still receive the benefits of batch normalzation but not severely negatively impact the speed of our learning. We found that this decrease in lag/increase in training speed made negligible impacts on the accuracy of our models but sped them up drastically (batch iteration speeds of ~8.5 seconds with default momentum vs ~6.5 seconds with the lowered momentum which is a very large difference when single epochs take about 3 hours to train. 

After the first dense layer (with its associated reshape layer and batch normalization), we add our first transposed convolution layer. A transposed convolutional layer is almost exactly what it sounds like. It is a convolutional layer that has its weight matrix transposed and this mathematical operation means that the convolutional layer no longer abstracts images into latent representations but instead takes in latent representations to create images of larger and larger size with every layer (aka upsampling). This is the only layer type that would work in this case because we are using 2D data with our images and the discriminator uses 2DConv layers to break down the images it is input. For these layers, we feed it the number of filters, the kernel size, strides, padding, and a standard deviation for our kernel initializer. For our generator, we chose a kernel size of 5, with strides of 2, and same padding. We selected a kernel size of 5 with strides of 2 for our generator mostly through trial and error. We originally tried 3 and 1 as we saw another similar network use those parameters but immediately found that the small kernel and tiny stride resulted in huge training time costs. We therefore shifted the kernel size up to 5 and increased the strides in turn but only by one and found that the training time became more manageable. The same padding is used because the images are already nicely cropped and resized meaning that no additonal resizing is needed and we want the output to have the same spatial dimensions as the input. These 2D Transposed Convolutional layers are followed by the leaky ReLU activation function with an alpha set to 0.2 (alpha has a default of 0.3). We decided upon a rectified linear units activation function because that is the best activation function for CNNs as they are extremely computationally efficient due to their very simple activation operation and do not suffer from the vanishing gradient issue of many other functions. We decided upon a leaky ReLU versus normal ReLU because we read online that ReLU can suffer from a "dying ReLU" issue which can happen when the ReLU consistently gets values under 0 which bricks the learning of the network. We found minimal differences in ReLU and LReLU when evaluating the time and accuracy of our models and therefore thought it would be best to go with LReLU to avoid overfitting by way of dying ReLU. We started with the default alpha of 0.3 but found that 0.2 slightly increased our accuracy for both models. This transposed layer is followed by the batch normalization. 

Each one of these transposed convolutional layers can be seen as upsampling the previous input into an image twice the size. This means that our dense layer takes in the noise vector and outputs a (4,4) image, the transposed convolutional layer with filters=filters turns that into (8,8), the filters//2 layer upsamples it to (16,16) and so on until the last transposed covolutional layer takes the image into its final (128,128,3) representation and uses the tanh activation function. We decided upon the tanh function because it runs faster than sigmoid, can handle negative values, and has decreased risk of vanishing gradient due to the (-1,1) range as opposed to sigmoid's (0,1). 

Overall, we progressively stepped up this generator from our first test run with output of (32,32,3) up to (64,64,3) and then finally to (128,128,3) because we felt that the increase in training time was well worth the increase in quality as the (32,32,3) was horrible quality, the (64,64,3) was alright but nothing near realistic and (128,128,3) offered a substantial increase over the (64,64,3). 

### create_discriminator:

This is the function we use to create our discriminator network. The summary for this architecture can be found in our paper. 

This model takes in images of size (128,128,3), downsamples them to a single output that classifies the image as a value between 0 and 1 to represent the 'realness' of the image as adjudged by the model. We feed it the initial filter value for the first convolutional layer, an alpha value for our LeakyReLU (previously discussed), a standard deviation for our kernel initializer (previously discussed), an input shape that represents the size of the images that are input to the model, the kernel size, strides, and padding for our convolutional layers and a momentum for the batch normalization (previously disccused). 

As the input is a plain image, the first layer is a convolutional layer that downsamples the (128,128,3) input image into a (32,32,3) abstraction. For these convolutional layers we started the discriminator with the same kernel size, strides, and padding as our generator just as a good general starting point. When we decided to change the kernel size and strides up, we matched that change with the discriminiator variables. After we ran that initially we attempted to tweak these hyperparameters to customize them for the discriminator. We found that decreasing the kernel size to 3 and the stride to 1 with a kernel size of 5 and stride of 2 in the generator maximized our accuracy but the return on marginal accuracy was not worth the extra training time. 

We decided upon a LReLU for the same reasons as discussed above. We have the first convolutional layer which takes it from (128,128,3) to (32,32,3) then the second convolutional layer is twice the filters which downsamples to (16,16,3) and so on until the last convolutional layer downsamples to (4,4,3) at which point we add a flatten layer. We originally did not have the flatten layer and received errors about input dimensions and dimensionality and quickly realized that in order to pass a single vector to the dense layer, we had to flatten the network so we did that as it was necessary for the model to work. 

After the flatten we have a dense layer that serves as the output of the discriminator, the 'realness' of a fed-in image. This output is a single number that is subjected to the sigmoid activation function. We used this activation function because we want a simple 0-1 output which is the range of the function. 

Much like the generator, we progressively stepped up the size of this network. We started at (64,64,3) and this worked well for the generator creating images of (32,32,3) and (64,64,3) but by the time we bumped the generator to (128,128,3) it no longer made sense to keep the discriminator at (64,64,3). When we added the additional layer the accuracy went up drastically, most likely as the discriminator was more 'evenly matched' with the generator. 

###create_DCGAN:

This is the relatively short function that instantiates a generator and discriminator based on the function calls above, compiles the discriminator, assembles the GAN from the generator and discriminiator, then compiles the GAN. 

Most of the variables fed into the DCGAN function are passed directly through to the generator and discriminator functions so I will only talk on the ones not previously mentioned. 
d_learning_rate is the learning rate for the discriminator's ADAM optimizer and d_beta is the beta_1 attribute of ADAM. We also have the same two variables for the generator. 

We chose to use ADAM for our optimizer for both the discriminator and the generator because ADAM is a optimization algorithm designed specifically for deep neural networks and often does quite well with GAN achitectures. We found that ADAM was faster than other optimazations like adagrad, sgd, and RMSprop and the accuracy was slightly better for ADAM but it was close enough where training time was the deciding factor. 

We originally decided on a generator learning rate of 0.001 and a discriminator learning rate of 0.0001 which worked well when the generator was creating images in (32,32,3) and (64,64,3) but we found that the generator learning rate of 0.001 was too high when we tweaked our model to have the generator make images in (128,128,3). After making adding the layer to the generator and noticing that the generator loss shot through the roof while the discriminator slowly decreased loss meaning that the discriminator was 'too good' and the generator actually got the best results in the first epoch and it diverged from there. When we lowered the learning rate of the generator to meet the learning rate of the discriminator, we found that this fixed the issue and the models no longer diverged. We then later went back and changed the discriminator learning rate to fall between our first two value attempts of 0.001 and 0.0001 by setting it to 0.0005 We originally completely ignored the beta values for both the discriminator and generator and used the default values of 0.9 but received nothing but pixelated images with no traces of a human face. After many uncesseful tweaks, we made it to the optimization function and tried feeding it alternative beta_1 and beta_2 values. We walked the beta_1 value down from the default of 0.9 to 0.3 and found that anything higher than 0.6 diverged the models and gave us bad results while anything lower than 0.4 had a similar end effect. We settled on 0.5 as it felt like a safe zone that gave us leeway on either side of the model divergence range of 0.4-0.6. We also attempted to mutate the beta_2 number in ADAM but found there to be no obvious effects of changing it from the default therefore we kept it as is. 

Once the DCGAN is successfully compiled, we returned the gan, the generator, and the discriminator. 



## Training Function

Our train function is essentially the main function of the program. All the networks are instantiated and all of the training and evaluation is done in it. 

Nearly all of the variables passed through on the train method are immediately passed into network instantiation and therefore they have been explianed above. The only exceptions are the batch size, eval size, epochs, and smooth. We experimented with batch sizes ranging from 16 to 256 and found that anything under 64 was unbearably slow for training. We were only able to run 1 epoch due to how long the training took at a batch size of 16 but it did not seem to make considerable imporovements on 64. We found that the models would diverge at 128 and 256 so we decided upon a batch size of 64. If we had unlimited time to train we may have gone with 32 but it did not seem to be a noticeable drop from 32 to 64. Our evaluation size was 16 and we honestly had no guidance about what we might want this to be and it was honestly inconsequintial on the actual performance of our model so we tried a few different results with no noticeable results. We ultimately went with an epoch of 3 but this was almost entirely based on time restraints (the final 3 epoch model took about 10 hours to train) and think that if we had a dedicated machine that was not shared, a higher epoch value could have helped improve quality of the models. The last new variable is smooth which we use for label smoothing when we train our discriminator on the real data. We do this label smoothing only on the discriminator handling the real data because we don't want the discriminator to become overconfident when adjudging the realness of the real images. If we do not give it this 'restriction on confidence' via the label smoothing, it will give overly confident guesses which could have a strong detrimental impact on the ability of the generator to create realistic images. This is because the discriminator giving an absolutely confident score for every real image would train the discriminator's weight to bias against the fake data, which is great for discriminator loss but not good for the generator, which is what we care about here. We researched what degree of smoothing is appropriate and found something above 0.8 is usually what is used, we tried .8 and .9 and found they both did drastically better than 1. We found that 0.9 helped the loss of the generator and discriminator diverge less than 0.8 so we went with 0.9. 

To start the training method, we created arrays of ones and zeros so that we could label all of our images either a 1 for real or a 0 for fake. ytr is the labels for the training set for the real data and ytf is the labels for the training set for the fake data. yer is the real evaluation labels and yef is the fake evaluation labels. We then call create_DCGAN to make the gan, generator (g), and discriminator (d). We then instantiate our loss array which we will add to later. 

We then jump into the actual training of the train method. We have a nested for loop with the outside tracking our epochs and the inside look going through each of the batches in the dataset. The for loop has the tqdm call around it which simply creates the loading bar shown as the network trains. We saw one of the networks online had used this and we found it to work extremely well as it took very little code to implement. Once inside the for loop, we immediately get the filenames for every image in the current batch from the x_train array. We then fill the xbr array with the actual images, for the first time using these instead of just the filenames. We obviously have to apply the preprocessing as well as the resizing and cropping that is done through the proprocess and load image methods. The necessity of these methods and actions can be found under the individual functions' descriptions in this readme. After the images for the batch have been pulled in and processed, we create a normally distributed latent space vector of dimensions=100. We were unable to find any concrete information on what type of initial distribution we should use for our noise vector but found that there made no real material difference whether we used normal or standard so we just used normal. We then created the batch of fake images with our generator model by calling the build in function, train_on_batch. This data is obviously necessary to have as is how we will train our generator (noise vector) and our discriminator (batch of real and fake images). 

We then set every layer in the discriminator to be trainable as we will start by training the discriminator independently of the generator. This makes sense as we want to have at least a semi-educated discriminator by the time it gets the first fake image or else the model would have no way to tell the fake from the real. We train the discriminator on our real batch (smoothed as discussed earlier) with the labels (all 1s) and the fake batch with the labels (all 0s). Once we have trained our discriminator on this batch, we set the discriminator layers to no longer be trainable. We do this because we next train the overall DCGAN which is composed of both the generator and the discriminator. We turned off the trainable attribute as we do not want the discriminator to train as we train the GAN, this is why we trained it beforehand. 

We then appended a simple if statement so that every 1000 batchs (or about 3x per epoch) the models are saved in a json format. This is simply a precaution as we had a few mistakes between epochs where our code would run a whole epoch but break at the end of the epoch before we could save the models. Therefore we added this so we would have models even from partial runs of epochs in case of unexpected error. This is the last operation from within the batch loop. The training is officially finished. 

Next we evaluate the model. As was discussed earlier, this is not extremely central to the results of our model as the subjective realism of the images is possibly more important than any validation numbers we get but we should still have the objective yard stick. We first generate the array of the filenames for our evaluation set and just as we did for the batches in training, we proprocess and load the images based on the filenames in our evaluation test. We then generate another latent space vector of size eval_size so that we can create fake images using our generator on the next line. This means that we now have a an array of the real validation images, an array of fake generated images, and we are ready to train the discriminator and gan for losses on these sets. We want to do this so that we can get a general feel of the losses of the discrimator versus the losses of the DCGAN so we can know how to best go about correcting and fine tuning the model. We first generate the discriminators loss from the real evaluation images, then the discriminators loss from the fake evaluatio images, and finally we evaluate the loss of the total gan model using the latent samples. We use these three test losses because we want to challenge the discriminator to tell between real and fake images but we also want to optimize the gan to create more accurate images. We then append these losses to our loss array declared before the loop. We again json-ize our models so that we have all three models saved at the end of every epoch in case something interesting happens between epoch iterations such as significant increases in loss. We then output a simple line that will serve as the output of our losses at the conclusion of every epoch so we can track progress. 

Finally we have the boolean images which when true, calls the show images function at the end of every epoch as well as show losses at the end of the entire train call as well as the show images for our selected sample from the completed model. These are so that we can get a subjective assessment of the accuracy of the model through the image generations while also quantitatively assessing the model through the losses chart. 



## Calling the Train Function

All of the code that actually creates the networks and starts the training process is the train function call at the end of the code. Here, all of the hyperparameter values can be seen and a short description of each variable can be found on the same line as the variable. We store the gan, discriminator, and generator as (gan, d, g). We then convert each network into a json format using the built in .to_json() method call then write a json file with the given using the json-ized networks. This is so we can use the saved models to generate new images or to train further. 

