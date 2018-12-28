---
layout: post
title: "Feature engineering"
subtitle: "Useful resources"
image: light-ninja.jpg
date: 2018-12-10
comments: true
---
# So you've built a feature store ... what next?

Now you have a feature store with hundreds (or thousands!) of columns - you'll likely need to prep them for inputs to train an ML model.

Here's the places I found useful when learning how to do this:

* [Tensorflow Feature Columns](https://www.tensorflow.org/guide/feature_columns)
* [Introduction to Embeddings](https://developers.google.com/machine-learning/crash-course/embeddings/video-lecture)
* [Embeddings in Keras](http://flovv.github.io/Embeddings_with_keras/)
* [Feature scaling](https://sebastianraschka.com/Articles/2014_about_feature_scaling.html)

So far, I've mainly been using the StandardScaler from sklearn along with the one hot encoding using pandas get_dummies.

```python
import pandas as pd
my_one_hot_category = pd.get_dummies(my_dataframe['my_category'])
my_dataframe.drop('my_category', axis=1, inpace=True)
```


This is because most of the categorical data I've used upto now is easily binned and can then be one hot encoded - without creating a large sparse vector to represent the category.

Embedding, feature hashing, and feature crossing are next on the list to try out.

# And remember...
Once you've trained your predictive model and carefully saved your model and its weights - you must also remember to ensure you can repeat any other feature preparation you've done.  In particular, make sure you've saved away your Scaler - so you can use it in your prediction deployment.

It took me a little time to figure out how to save a Scaler object - here's how I'm doing it - hope it's right!

```python
from sklearn.preprocessing import StandardScaler
scaler = StandardScaler().fit(X)

from sklearn.externals import joblib
joblib.dump(scaler, "scaler.save")

# And then load with:
scaler = joblib.load("scaler.save")
```

# Lessons learned the hard way
Here's a few mistakes made along the way - I made them so you don't have to!

**fillna(0)** - don't be too trigger-happy with replacing nulls with 0.  Replacing with the column mean will often make more sense ... blindly filling missing data with zero can skew your model.

**Know Your Features** - after being super-stoked with a high-accuracy, high-precision, and high-recall trained model - I subsequently found that I'd left in a column which was a dependent variable :(   Always be suspicious of good results!

**RandomizedSearchCV()** - is awesome, make sure you know how to use it.   You can use this method to optimise accuracy, precision, or recall through k-fold cross validation - then test the best parameters on the held-back test set.

**Less is more** - _sometimes_, your model will generalise better with less training epochs, ie, less likelihood over overfitting to the training data.