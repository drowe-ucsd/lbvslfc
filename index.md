<head>
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/katex@0.15.1/dist/katex.min.css" integrity="sha384-R4558gYOUz8mP9YWpZJjofhk+zx0AS11p36HnD2ZKj/6JR5z27gSSULCNHIRReVs" crossorigin="anonymous">
  <script defer src="https://cdn.jsdelivr.net/npm/katex@0.15.1/dist/katex.min.js" integrity="sha384-z1fJDqw8ZApjGO3/unPWUPsIymfsJmyrDVWC8Tv/a1HeOtGmkwNd/7xUS0Xcnvsx" crossorigin="anonymous"></script>
  <script defer src="https://cdn.jsdelivr.net/npm/katex@0.15.1/dist/contrib/auto-render.min.js" integrity="sha384-+XBljXPPiv+OzfbB3cVmLHf4hdUFHlWNZN5spNQ7rmHTXpd7WvJum6fIACpNNfIR" crossorigin="anonymous"
    onload="renderMathInElement(document.body);"></script>
</head>

<h1> Introduction </h1>
This project is based on the paper <a href="https://cseweb.ucsd.edu//~viscomp/classes/cse274/fa21/readings/a193-kalantari.pdf">Learning-Based View Synthesis for Light Field Cameras</a> by Kalantari et al.  In this paper, the authors use a 2-stage neural network to synthesize images from non-input view directions in a light field camera.  In theory, this work could be used to increase the spatial resolution of consumer light field cameras while retaining the same angular resolution, by simply synthesizing the intermediate views.

<p style="text-align:center;">
<video width="541" height="376" controls loop autoplay muted>
  <source src="flfinal.mp4" type="video/mp4"/> 
</video>
</p>

<h1> Overview of the Original Paper </h1>
Below, I give a brief synopsis of how this paper works.

<h2> Architecture </h2>
The neural network architecture in this paper consists of two CNNs, each with 4 convolutional layers and no fully-connected layers.  The network takes in the 4 corner views of an 8x8 angular resolution grid, as well as the \\( (u, v) \\) coordinates of the desired input view.  The full network learns to directly synthesize an image from the desired input view.  In all results shown here (and most in the original paper), there is a square in the upper left corner that indicates which view the network is synthesizing (in light gray) and which views the network took as input from the light field image (in orange).

<p style="text-align:center;"><img src="viewlayout.png" width=215 height=215/></p>

<h3> Disparity Estimator </h3>
The first CNN is called a "disparity estimator", and it aims to roughly estimate the disparity (a measure of the "motion" or "distance" between pixels in two camera views) as a 1-channel image.  Its input is 200 image features which the authors automatically create before applying the network.  These features are intended to help the network see the disparity at each point in the image.  First, the authors <i>backward warp</i> the 4 input images (converted to grayscale); this step shifts each of the pixels in each input image by a predefined disparity level \\( d_l \\).  Thus they construct a backwarped image \\( \bar{L}\_{p\_i}^{d\_l}(s) = L\_{p_i}\[s + (p_i - q)d_l\] \\), where \\(p_i\\) and \\( q \\) are the \\( (u, v) \\) coordinates of the input and target views, respectively.  They perform this for each of the 4 input views, and warp using 100 predefined disparity levels between -21 and 21.  Then, 100 features are found by averaging over the 4 views at each disparity level, and 100 features are found by taking the standard deviation of the 4 views at each disparity level.  Finally, the authors stack the 200 features into a 200 x h x w tensor.  Below, I show the disparity mean and standard deviation features for each disparity level.  Notice that at different points in the scene, the videos line up at different disparity levels.  This information can be used by the network to determine the disparity at those points.

<p style="text-align:center;">
<table>
  <tr>
    <td>
      <video width="361" height="251" controls loop autoplay muted>
        <source src="dispfeat33.mp4" type="video/mp4"/> 
      <!-- Your browser does not support the video tag. -->
      </video>
    </td>
    <td>
      <video width="361" height="251" controls loop autoplay muted>
        <source src="dstdev33.mp4" type="video/mp4"/> 
      <!-- Your browser does not support the video tag. -->
      </video>
    </td>
  </tr>
</table>
</p>

I also show the output of the disparity network for a single view here.  Note that in this disparity output, close objects are bright and far objects are dark.  This is because closer objects shift more between the 4 views, and further objects shift less.  This is a good sign (during implementation) that the network is able to handle objects at different depths.

<p style="text-align:center;"><img src="output_disp.png"/></p>

<h3> Color Predictor </h3>
The color predictor network uses the disparity estimator output, the 4 input images, and the \\( (u, v) \\) coordinates of the target view to determine the 3-channel output view.  To create the features for this network, the authors use the disparity estimator output to <i>forward warp</i> the 4 (color) input views.  This is given by the equation \\( \bar{L}\_{p_i}(s) = L_{p_i}\[s + (p_i - q)D_q(s)\] \\), where \\( D_q(s) \\) gives the disparity estimator's output at pixel \\(s\\).  The warped views are then concatenated with the \\( (u, v) \\) target view coordinates as well as the disparity output, giving a \\( 3*4 + 2 + 1 = 15 \\) channel input.  The layers in this step are the same as in the disparity estimator, but the input layer takes in 15 channels rather than 200.  Putting these details together, here is the final network:

<p style="text-align:center;"><img src="network.png"/></p>

At one point in my implementation I implemented the forward warping incorrectly between the networks (I swapped angular coordinates in some places but not in others), and it was clear from the disparity output, because the first network was essentially learning the identity mapping, but had a few artifacts on the edges of objects.  I show this buggy output below.

<p style="text-align:center;"><img src="bad_disp.png"/></p>

<h2> Other Paper Details </h2>
In the original paper, training occurs on 60x60 patches, with batch size 20.  They use the L2 loss, and use ADAM to update the weights of the network.  Additionally, since the warp operations will likely try to read values between pixels if implemented naively, they use bicubic interpolation to sample the all images that are warped (forward or backward).  Backpropagation through the bicubic interpolation step during the forward warp is performed numerically.

<h1> My Implementation </h1>
I implemented this paper for my final project in CSE 274 Fall 2021: "Sampling and Reconstruction of Visual Appearance: From Denoising to View Synthesis".  Here, I detail my implementation and how I got there.
<h2> Early Steps </h2>
Earlier in the quarter, I was interested in implementing <a href="https://cseweb.ucsd.edu//~viscomp/classes/cse274/fa21/papers/nex-cvpr21.pdf">NeX: Real-time View Synthesis with Neural Basis Expansion </a> (Wizadwongsa et al. 2021).  However, after working on the implementation for a while, I realised that given my resources and time, it would be too complicated for me to implement and run in a single quarter.  I started looking at the other papers covered in this course, and briefly played with the idea of implementing one of the denoising papers, but decided to instead implement this learning-based view synthesis paper from 2016, as it fell within my interests and matched the resources available to me.  With that, I began implementing the network and finding solutions that allowed me to access school GPUs.

<h2> Frameworks / Resource Acquisition </h2>
Our course gives access to UCSD DataHub, a service which allows students to connect to computers with decent GPUs and pre-installed machine learning environments.  Through this service, I was able to use a 2080 Ti, accessing it through an online Jupyter Notebook.  I used Pytorch (with CUDA) for implementation and training of the network, as well as NumPy for a few transformations of the data that aren't required to be differentiable.  I also used cv2 (OpenCV) for writing my video results.  It took quite a while, but I was also able to convince the kind folks at DataHub to place my ~30gb worth of data on their computers, unzip the folders, and give me access to them.

<h2> Differences from the Original Paper </h2>
My implementation differed from the original paper slightly, as it used different frameworks and had access to different resources.  I summarize the differences below:
<ul>
  <li>In the original paper (as far as I could tell), patches are generated before training and inserted into local folders.  This allows the batches to be less correlated -- a single batch can have patches from many images with many input views.  However, due to the limitations of DataHub, I found that this would be infeasible.  As a student, I only have access to ~10gb of local storage, and ~16gb of RAM, so it would neither be possible to push all the patches to new files nor to construct uncorrelated batches from multiple images and views.  Thus, in my implementation, I load a single light field image at a time and train on patches exclusively from that image with a single target view.  This is a significant batch correlation, which likely decreased my convergence rate, but I hope I can convince you that I still generate decent results.</li>
  <li>I used Pytorch (with CUDA) for implementation and training, whereas the original paper authors used MATLAB with MatConvNet</li>
  <li>I used uniform He initialization (assuming ReLU activations) rather than Xavier initialization (which the authors use).  I made this choice after some short research that informed me that Xavier is optimized for sigmoid activations rather than ReLU activations.</li>
  <li>I likely trained the network for less time.  Although I used the DataHub GPU for ~2 days, it seems that training is slower during peak hours.  This means that my access to the GPU is mediated by other students' accessing it as well, and thus I likely did not train for the same effective GPU time as in the paper.</li>
</ul>

<h2> Training </h2>
I trained for a few days, until the network seemed to have reached (and passed) its lowest test loss (I use the minimum test loss weights for all non-bug results shown).  The train/test loss graphs are shown below.  Note that the scales are different -- for the training loss, I did not normalize by the number of samples, and the training loss is sampled more frequently than the test loss, but the progress of the curve is nonetheless clear.

<p style="text-align:center;"><img src="losses.png"/></p>

<h2> Other Implementation Details </h2>
In the results I presented in class, I mentioned that I was having issues with low saturation in my output images.  After a student commented that this could likely be a bug related to gamma correction, I looked at the paper authors' code to find out what type of tone correction I would have to apply.  The result is that my results were mostly correct in my presentation, but indeed needed to be gamma corrected by \\( \gamma = 1.5\\) and also saturation-scaled by 1.5.  Thus my images in this report are more color-correct than those from my presentation.  In addition, the images shown in the results section are from versions of the network trained further than the images in the presentation, and thus there will likely be fewer artifacts overall.  Below I show the desaturated flower video I presented, for reference; you'll find the improved results further down.

<p style="text-align:center;">
<video width="541" height="376" controls loop autoplay muted>
  <source src="flviews2.mp4" type="video/mp4"/> 
</video>
</p>

For the forward and backward image warps, I utilized Pytorch's built-in meshgrid and grid_sample functions.  This allowed me to create a uniform grid of \\( (x, y) \\) points, add the necessary offsets to each coordinate based on the type of warp, and then sample the bicubic interpolated input images using the determined grid.  Additionally, since both of these are Pytorch functions, I was able to use Pytorch's default gradients for each of these functions, thus abstracting away the messy technical details involved in finding numerical gradients for bicubic interpolation.

<h1> Results </h1>
Finally, let us examine the results.  First, here are the videos of my network cycling through every synthesized input view on a flower light field image.

<p style="text-align:center;">
<video width="541" height="376" controls loop autoplay muted>
  <source src="flallfinal.mp4" type="video/mp4"/> 
</video>
</p>

I also show results circling around the edges for several more light fields.  Note that for some scenes, the occlusion is too difficult (here, the brush scene fails to give a convincing background), likely due to the difficult occlusion boundary.

<p style="text-align:center;">
<table>
  <tr>
    <td>
      <video width="361" height="251" controls loop autoplay muted>
        <source src="bffinal.mp4" type="video/mp4"/> 
      <!-- Your browser does not support the video tag. -->
      </video>
    </td>
    <td>
      <video width="361" height="251" controls loop autoplay muted>
        <source src="shfinal.mp4" type="video/mp4"/> 
      <!-- Your browser does not support the video tag. -->
      </video>
    </td>
  </tr>
  <tr>
    <td>
      <video width="361" height="251" controls loop autoplay muted>
        <source src="tbfinal.mp4" type="video/mp4"/> 
      <!-- Your browser does not support the video tag. -->
      </video>
    </td>
    <td>
      <video width="361" height="251" controls loop autoplay muted>
        <source src="rxfinal.mp4" type="video/mp4"/> 
      <!-- Your browser does not support the video tag. -->
      </video>
    </td>
  </tr>
</table>
</p>

In the original paper, there are also decent results for extrapolated views, though these tend to have lots of artifacts.  Here, I show some extrapolated results for my implementation of the network, which has even further artifacts than the paper (and cannot really handle views outside its main range), suggesting that my implementation has strongly overfit to the original window coordinates.

<p style="text-align:center;">
<video width="541" height="376" controls loop autoplay muted>
  <source src="flextfinal.mp4" type="video/mp4"/> 
</video>
</p>

Finally, I show results for images taken on a cellphone.  Since the original light field camera has a very small baseline and the views are very well-calibrated, it is clear that using cellphone images will not get perfect results on a network trained on the light field images.  However, I did not expect the results to be this bad.  Nonetheless, let this be a lesson to anyone trying to train on calibrated images and test on uncalibrated (and unwarped) ones.

<p style="text-align:center;">
<video width="480" height="354" controls loop autoplay muted>
  <source src="blphonefinal.mp4" type="video/mp4"/> 
</video>
</p>

<h2> Conclusion </h2>
I had a fun time implementing this paper, and I learned quite a bit about view synthesis and neural network implementation.  It was especially interesting to me to see places where the original paper may not hold up as well (e.g. the toilet brush scene above, or the phone images), since these results are often left out.  I am proud of my network's results, especially considering the storage/memory/execution limitations imposed by DataHub, and I think my experience will help me in implementing and understanding future papers related to light field imaging and view synthesis.

<h2> Glitch Images </h2>
As extra content, I leave you with some of my favorite glitchy videos from when I was trying to debug my network.
<p style="text-align:center;">
<table>
  <tr>
    <td>
      <video width="361" height="251" controls loop autoplay muted>
        <source src="dispfeat3.mp4" type="video/mp4"/> 
      <!-- Your browser does not support the video tag. -->
      </video>
    </td>
    <td>
      <video width="361" height="251" controls loop autoplay muted>
        <source src="blglitch.mp4" type="video/mp4"/> 
      <!-- Your browser does not support the video tag. -->
      </video>
    </td>
  </tr>
</table>
</p>

