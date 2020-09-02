---
layout: post
title: Spam Detection Microservice Using Snorkel (9/2/20)
---
This post walks through the process of putting a spam detection model using [Snorkel](https://github.com/snorkel-team/snorkel) into a microservice using Flask, Docker, and AWS.

## Post Outline
- [Spam Detection using Snorkel](#spam-detection-using-snorkel)
- [Interfacing with Snorkel using Flask](#interfacing-with-snorkel-using-flask)
- [Deployment](#deployment)
- [Code](#code)
- [Resources](#resources)

## Spam Detection using Snorkel
We will be deploying a model that classifies a comment as spam or not spam.
Snorkel can be used for this task.
At a high level, we have a dataset of YouTube comments.
We add labeling functions, which are Python classes that classify a comment as spam, not spam, or abstained.
These labeling functions can range from keyword labeling, which labels a comment based on whether it contains a particular word, to third party models such as [TextBlob](https://textblob.readthedocs.io/en/dev/index.html), which identifies sentiment.
The sentiment can then be used to classify a comment if it is believed that there is a correlation.

Here is an example labeling function that classifies a comment as spam if it contains the word `check`.
```
@labeling_function()
def check(x):
    return SPAM if "check" in x.text.lower() else ABSTAIN
```

We then combine these different labeling functions to generate probabilistic labels, where each input has a corresponding vector of size 2 (spam or not spam).
The first index corresponds to the probability that it is spam and the second contains the probability that it is not spam.
This combining, which resolves overlaps, can be done using majority voting or Snorkel's own [LabelModel](https://snorkel.readthedocs.io/en/master/packages/_autosummary/labeling/snorkel.labeling.model.label_model.LabelModel.html#snorkel.labeling.model.label_model.LabelModel).

We then use these probabilistic labels to create labeled data by assigning to each input the class that had the higher probability. 
This labeled data can then be used to train a discriminative model to make predictions beyond the given dataset.
For example, we can use Logistic Regression.

The above is a quick summary of Snorkel's [post](https://www.snorkel.org/use-cases/01-spam-tutorial#-snorkel-intro-tutorial-data-labeling).

## Interfacing with Snorkel using Flask
I wanted the user to be able to perform two actions: getting a prediction for an input comment and adding a new labeling function.
To do this, I refactored the Snorkel tutorial code into a class `SpamDetection` defined in [`spamdetection.py`](https://github.com/andrew128/spam-detection-microservice/blob/master/app/snorkel_spam_detection/spamdetection.py).

The class automatically loads the training data and trains upon initialization.
The class also has three member functions, `addLfs()` for adding a new labeling function, `train()`, and `predict()`.
We define all the initial labeling functions in `labelingfunctions.py`.

In the [`controller.py`](https://github.com/andrew128/spam-detection-microservice/blob/master/app/controller.py) file, the routing is defined.
Upon visiting the page, users are presented with two forms: one for making a prediction and one for adding a new labeling function.
When the user submits a comment for prediction, the model is run and the prediction is displayed on the same page.
When a user submits a word into the new labeling function, the model will add that labeling function (which classifies the word as spam), to its own set of labeling functions. 

## Deployment
There is a Dockerfile and a Docker Compose file to deploy the Flask app.
We can use Docker's ecs plugin to deploy to AWS.
As of writing this post, the plugin is only available in Docker's `edge` channel.
Using the plugin consists of a few basic docker commands and an AWS account.
Here is a [link](https://github.com/docker/ecs-plugin/tree/master/example) for a complete example and walkthough.

## Code
[Here](https://github.com/andrew128/spam-detection-microservice) is the full repo.

In the root directory is the `Dockerfile` and the `docker-compose.yml`, which define the docker service.
The `app/` directory contains the Flask app.
It uses the MVC design pattern.
The file `controller.py`, which contains the definitions for the Flask routes, uses the Flask Form classes defined in `model.py` to display the front page defined in `templates/view.html`.
The directory `snorkel_spam_detection` holds the Snorkel model.

## Resources
- [Snorkel Spam Detection Tutorial](https://www.snorkel.org/use-cases/01-spam-tutorial#task-spam-detection)
- [Deploying a Docker container to AWS](https://github.com/docker/ecs-plugin/tree/master/example)