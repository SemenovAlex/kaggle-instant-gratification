# Instant Gratification Solution (47th place)

I entered the Instant Gratification competition when it was 16 days to go, so there was already a lot of important information available in Kernels and Discussions. Shortly, in the Instant-Gratification competition we deal with a dataset of the following structure:
    
- [Data seemed to be generated by groups](https://www.kaggle.com/c/instant-gratification/discussion/92930#latest-541375) corresponding to the column **wheezy-copper-turtle-magic**, which means that we have 512 independent datasets of the size approximately 512 by 255 for training our models.

- Out of 255 features [only features with high variance supposed to be important](https://www.kaggle.com/fchmiel/low-variance-features-useless/). The number of important features varies from 33 to 47.

- [QDA proved to be a good modeling approach](https://www.kaggle.com/speedwagon/quadratic-discriminant-analysis) for this case and can give AUC of 0.966. 

- This high quality can be explained by the fact that the [data probably was generated with make_classification function](https://www.kaggle.com/mhviraf/synthetic-data-for-next-instant-gratification).

Knowing that let's look closer at what make_classification function does. This function has the following parameters:

- n_samples: The number of samples.
- n_features: The total number of features.
- n_informative: The number of informative features.
- n_redundant: The number of redundant features, linear combinations of informative features.
- n_repeated: The number of duplicated features.
- n_classes: The number of classes.
- n\_clusters\_per_class: The number of clusters per class.
- weights: The proportions of samples assigned to each class.
- flip_y: The fraction of samples whose class are randomly exchanged.
- class_sep: Larger values spread out the clusters/classes and make the classification task easier.
- hypercube: Parameter corresponding to the placement of the cluster centroids.
- shift: Shift of the feature values.
- scale: Scaling of the features by the specified value.
- shuffle: Shuffling the samples and the features.
- random_state: Random state.

So, I invite you to follow the sequence of my thoughts.

***

### Step 1. Size and features (first 6 parameters of make_classification function).

From the data structure (512 groups and 255 variables) and size (262144 = 512 * 512), we can assume that **n_features=255** and **n_samples=1024** were taken to generate both training and test data sets (both private and public).

Now, let's talk about features. First of all, repeated features are exact copies of important features, and because we don't see such columns in competition data we'll set **n_repeated=0**. Let's generate data set with make_classification function and following parameters:

~~~~
fake_data = make_classification(
    n_samples=1024, 
    n_features=255, 
    n_informative=30, 
    n_redundant=30, 
    n_repeated=0, 
    n_classes=2, 
    shuffle=False)
~~~~

By setting shuffle=False, I force first 30 columns to be informative, next 30 to be redundant and others to be just noise. By doing it 1000 times lets look at the distribution of the standard deviation of the informative, redundant and random features.

<a href="https://ibb.co/HG8WWDR"><img src="https://i.ibb.co/L97LLzB/features-std.png" alt="features-std" border="0" width="1000px"></a>

We see clear difference in standard deviation for informative, redundant and random features, that's why selecting important features with 1.5 threshold works so well. Moreover, there are no features in the competition data that have std bigger than 5, which leads us to an assumption that **n_redundant=0**

Number of important features ranges from 33 to 47, so **n_informative** in **{33,...,47}**. Number of classes is obviously **n_classes=2**. 

***

### Step 2. Shift, Scale and randomness (last 4 parameters of make_classification function).

Because mean and standard deviation for random columns are 0 and 1 respectively, we can assume that shift and scale were set to default **shift=0**, **scale=1**. On top of that, standard deviation for important columns also look the same for competition data and for data that we've just generated. So it's a pretty promising assumption and we can go further. Parameters **shuffle** and **random_state** are not so important, because they not change the nature of the data set.

***

### Step 3. Most interesting parameters (why QDA is not so perfect?).

Parameters **n_clusters_per_class, weights, class_sep, hypercube** are the ones that we are not very sure about, especially **n_clusters_per_class**. Weights look to be the same, because target seemed to be balanced. What makes it imposible to get 100% accurate model is that **flip_y>0**, and we can not predict this random factor in any case. However, what we can try is to build a model that can perfectly predict when **flip_y=0** and leave the rest for the luck.

QDA shown to be a very good approach, but let me show you the case when it's working not so good:

<a href="https://ibb.co/vVDRDf7"><img src="https://i.ibb.co/k5MvMzR/4-components.png" alt="4-components" border="0" width="1000px"></a>

Data set was generated with make_classification:

~~~~
make_classification(
    n_samples=1000, n_features=2, n_informative=2, n_redundant=0, 
    n_classes=2, n_clusters_per_class=2, flip_y=0.0, class_sep=5,
    random_state=7, shuffle=False
)
~~~~

Now, lets look how QDA algorithm can hadle it:

<a href="https://ibb.co/pWRTdtD"><img src="https://i.ibb.co/RhBKcxn/qda-model.png" alt="qda-model" border="0" width="1000px"></a>

Pretty good, however, we see that it doesn't see this structure of four clusters in the data. From make_classification documentation one can find that these clusters are gaussian components, so the best way to find them is to apply Gaussian Mixture model. 

_Note: Gaussian Mixture model will only give you clusters, you need to bind these clusters to one of the classes 0 or 1. That you can do by looking at the proportion of the certain class in each cluster and assigning cluster to the dominant class._

Lets look how GMM algorithm performed at this data:

<a href="https://ibb.co/Q9sPPxm"><img src="https://i.ibb.co/0qPCCzJ/gm-model.png" alt="gm-model" border="0" width="1000px"></a>

Nearly perfect! Let's move to the step 4.

***

### Step 4. Effect of flipping the target

Knowing that we can build nearly the best classifier, what is the effect of flipping the target? Suppose that your classes are perfectly separable and you can assign probabilities that will lead to a perfect AUC. Let's see an example of 10000 points, 5000 in each class:

Classes = \[0, 0, ..., 0, 1, ..., 1, 1\]

Predictions = \[0.0000, 0.0001, ..., 0.9998, 0.9999\]

Lets flip the target value for 2.5% (250) of the points (1) in the middle, (2) on the sides, (3) randomly, and look at the AUC:

<a href="https://ibb.co/M1cNwkw"><img src="https://i.ibb.co/c86Tfhf/flipping-effect.png" alt="flipping-effect" border="0" width="500px" align="center"></a>

We see that the impact from flips can drammatically change the result. However, let's face two facts:

1. We can not predict this randomness.
2. We can assume that the flips are evenly spread across our predictions.

Let's also look at the distribution of AUC for randomly flipped target.
