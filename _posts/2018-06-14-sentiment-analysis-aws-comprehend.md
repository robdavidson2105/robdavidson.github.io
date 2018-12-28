---
layout: post
title: "Sentiment Analysis"
subtitle: "Using AWS Comprehend"
image: emojis.jpg
date: 2018-06-14
comments: true
---
# Sentiment Analysis
So let's say you want to analyse the sentiment of your customers' forum posts, or emails, or feedback reviews, ... or anything text based.

You could spend some time developing your own model - or using pre-trained open-source models such as [NLTK](https://www.nltk.org/).   *AND* in order to learn and understand the subject in more depth then you really ought to do this.

Then once you're happy that you know what you're doing - then for a number of use-cases you can now switch to using AWS Comprehend.  It's pre-trained, it's available as API, it's scalable - so, as ever, Amazon have done the 'undifferentiated heavy-lifting'.   I'm not quite sure it's fair to say 'undifferentiated' in this subject area just yet - but 'heavy lifting' for sure.

# AWS Comprehend
Let's take a look at the [Comprehend console](https://aws.amazon.com/comprehend/) - you can set up an AWS account and get a year for free - as long as you stay under the usage limits.  For your own lab environment - you'll be hard-pushed to break into the paid-for service.

At the comprehend console page, you can type in some example text ...

___

![Fig1](/assets/images/comprehend1.png){:class="img-fluid"}

**_Fig 1_**

___

... and then press the 'Analyze' button.

The first thing you'll see is an Entity analysis - in this example, it has correctly identified 'Amazon' as an Entity within the text.

___

![Fig2](/assets/images/comprehend2.png){:class="img-fluid"}

**_Fig 2_**

___

Now take a look at the key phrases tab.  The key phrases have been underlined in the 'Analyzed text' pane - in this example it's not done too well ... I think it should have at least got 'machine learning' as a key phrase!   But you can be sure that Amazon are continuing to train their model - and it will get better and better.

___

![Fig3](/assets/images/comprehend3.png){:class="img-fluid"}

**_Fig 3_**

___

The sentiment tab shows the scores associated with positive, negative, neutral, and mixed sentiment.  It's fared much better here - and I've found that AWS Comprehend is generally pretty good at predicting sentiment.

___

![Fig4](/assets/images/comprehend4.png){:class="img-fluid"}

**_Fig 4_**

___

# Create an API
Now let's create an API that we can use within an application.

The first thing to do is create a lambda function - this isn't a lambda walkthrough - just Google it!  But here's the outline of our lambda routine.

___

![Fig5](/assets/images/comprehend5.png){:class="img-fluid"}

**_Fig 5_**

___

And the lambda itself only needs a few lines of code to take a json input of the form:

```json
{"review": "Analyse this!"}
```

Here's the code which will return the results from the Comprehend API.

```python
import boto3
import json

comprehend = boto3.client(
    service_name='comprehend', 
    region_name='xxxxx'
    )
def lambda_handler(event, context):
    text = format(event['review'])
    jsonstr = comprehend.detect_sentiment(
        Text=text, 
        LanguageCode='en'
        )
    return  {
        'statusCode': '200', 
        'body':  jsonstr, 
        'headers': {'Content-Type': 'application/json'}
        }
```

Finally you just need to configure an API Gateway (again Google it) and we're done!

___

![Fig6](/assets/images/comprehend6.png){:class="img-fluid"}

**_Fig 6_**

___

The charges for AWS Comprehend are tiny - given they've done all the training and host the infrastructure - it's just not worth doing it yourself (imho)!
