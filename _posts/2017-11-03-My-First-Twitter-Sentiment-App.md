---
category: Blog
---
# Twitter Sentiment Analysis

## Lexicon-based Methods
After doing some research, I discovered that the most common method for performing sentiment analysis of text is through lexicon-based methods. A lexicon is a list of words that is scored in some way based on the emotional content of the word. The most simple lexicons score words in a binary way, such as postitive / negative ('bing' lexicon from Bing Lui and collaborators). Other lexicons may assign a numerical score to indicate not only the direction but the intensity of the emotion ('afinn' lexicon from Finn Arup Nielsen). Other lexicons will also go beyond positive / negative to assign words to a more specific emotion, such as surprise or anger ('nrc' lexicon from Saif Mohammad and Peter Turney).

Lexicons can be further categorized by the domain (such as community-specific lexicons for subreddits) and by the time period (a sentiment analysis of Shakespeare should not use a lexicon based on the modern use of the English language). If you are interested for these more unique lexicons, see this [Stanford Repository](https://nlp.stanford.edu/projects/socialsent/).

#### Data Prep
For any of the methods, we are going to need a data set to analyze. For this assignment, I was particularly interested in looking at the sentiment of tweets related to LADOT and Vision Zero. 

For the analysis, I am going primarily use R. For an introduction to text analysis with R, I recommed reading ["Text Mining with R"](http://tidytextmining.com/), which is freely available online. 

I am going to do all of the lexicon-based methods of sentiment analysis using the R package 'Syuzhet', which includes all of the main lexicons mentioned in the introduction. For an introduction to using Syuzhet, see [this vignette](https://cran.r-project.org/web/packages/syuzhet/vignettes/syuzhet-vignette.html).

#### Bing Lexicon 

#### NRC Lexicon 
Syuzhet includes the [NRC Word-Emotion Association Lexicon](http://saifmohammad.com/WebPages/NRC-Emotion-Lexicon.htm) algorithm from Saif Mohammad and Peter Turney. As I mentioned in the introduction to lexicon-based methods, the NRC lexicon goes beyond positive/negative to include scores these 8 more specific sentiments: anger, anticipation, disgust, fear, joy, sadness, surprise, and trust. 

For this method, I loosely followed the methodology published by [Julia Silge](https://juliasilge.com/blog/joy-to-the-world/) in the analysis of her own tweets. Julia Silge performed an analysis of a data dump of her own tweets. For this project, I'm going to use the new(ish) 'rtweet' package in R. According to [this blog post](https://www.r-bloggers.com/a-call-to-tweets-blog-posts/), the rtweet package is a more intuitive way to work with the twitter data.

I wanted to explore the other 8 emotions in the NRC Lexicon Library. 

#### Create a Twitter App
For any of the methods, we are going to need to create a twitter 'app' that will provide the authentication for us to access the basic REST API service. Navigate to [twitter](http://apps.twitter.com) and fill out the app name, description, and URL for your basic app. Following the guidance from [rtweet package here](http://rtweet.info/), make sure to set the set the following for the callback field:
```
http://127.0.0.1:1410
```

## Machine Learning Methods
Another method that I wanted to try included (this method using doc2vec)[https://www.r-bloggers.com/twitter-sentiment-analysis-with-machine-learning-in-r-using-doc2vec-approach/?utm_source=feedburner&utm_medium=email&utm_campaign=Feed%3A+RBloggers+%28R+bloggers%29]. The previous approach that the author took was a word-based approach, where the final 'sentiment' is calculated based on the difference of positive and negative words. The author updated the approach with a machine-learning method with a training set where the tweets were classified in their entirety.

However, I lowered my expectations after reading some of the comments on the original blog post. I found the following comment particularly insightful:
```
Hi Sergey

I came across your blog post at R-Bloggers and decided to play a bit with the code you shared. 
First of all thanks for sharing it and writing this, it is a great help to less proficient R users like me. 
I have tried to apply the code to couple subject areas and got the results I want to share:
- When I've used keyword from my business area - name of eCommerce platforms like Magento, Shopify and Hybris the average sentiment score was suspiciously similar, around 0.66 
- Then I've tried to apply the code to a more conventional topic and used keywords "trump" and "#trump". The resulting analysis shows much more positive attitude to Mr.Trump than I expected (image attached)
```
and then another commenter...
```
I agree with Alex's assessment. I obtained far more accurate results using other R packages like syuzhet,sentimentr and RSentiment which do not require any training but leverage Lexicon based techniques instead. I think such methods are better suited towards the semantics of tweets which require more sophisticated methods like valence shifting. However, I believe the use of CNN's treating text as "images" seems far more promising than text2vec in the world of deep learning and NLP. Just my 2 cents.
```
