

```python
from tweepy import API
from tweepy import Cursor
from tweepy.streaming import StreamListener
from tweepy import OAuthHandler
from tweepy import Stream
import numpy as np
import pandas as pd

import twitter_credentials
```


```python
#### TWITTER AUTHENTICATOR ####
class TwitterAuthenticator():
    
    def authenticate_twitter_app(self):
        auth = OAuthHandler(twitter_credentials.CONSUMER_KEY, twitter_credentials.CONSUMER_SECRET)
        auth.set_access_token(twitter_credentials.ACCESS_TOKEN, twitter_credentials.ACCESS_TOKEN_SECRET)
        return auth
```


```python
#### TWITTER STREAM LISTENER ####
class TwitterListener(StreamListener):
    """
    This is a basic listener just prints recieved tweets
    """
    def __init__(self, fetched_tweets_filename):
        self.fetched_tweets_filename = fetched_tweets_filename
        
    def on_data(self, data):
        try:
            with open(self.fetched_tweets_filename, 'a') as tf:
                tf.write(data)
            return True
        except BaseException as e:
            print('Error on data %s' % str(e))
        return True
    
    def on_error(self, status):
        if status == 420:
            # Returnig False on data_method in case rate limit occurs
            return False
        print(status)
```


```python
#### TWITTER STREAMER ####
class TwitterStreamer():
    """
    class for sreaming and processing live tweets 
    """
    def __init__(self):
        self.twitter_authenticator = TwitterAuthenticator()
        
    def stream_tweets(self, fetched_tweets_filename, hash_tag_list):
        # This handles Twitter authentication and the connection to Twitter streaming API.
        listener = TwitterListener(fetched_tweets_filename)
        auth = self.twitter_authenticator.authenticate_twitter_app()
        stream = Stream(auth, listener)
        
        # This line filter Twitter streams to capture data by the keywords
        stream.filter(track=hash_tag_list)
```


```python
#### TWITTER CLIENT ####
class TwitterClient():
    
    def __init__(self, twitter_user=None):
        self.auth = TwitterAuthenticator().authenticate_twitter_app()
        self.twitter_client = API(self.auth)
        self.twitter_user = twitter_user
        
    def get_twitter_client_api(self):
        return self.twitter_client
    
    def get_user_timeline_tweets(self, num_tweets):
        tweets = []
        for tweet in Cursor(self.twitter_client.user_timeline, id=self.twitter_user).items(num_tweets):
            tweets.append(tweet)
        return tweets
    
    def get_friend_list(self, num_friends):
        friends_list = []
        for friend in Cursor(self.twitter_client.friends, id=self.twitter_user).items(num_friends):
            friends_list.append(friend)
        return friends_list
    
    def get_home_timeline_tweets(self, num_tweets):
        home_timeline_tweets = []
        for tweet in Cursor(self.twitter_client.home_timeline, id=self.twitter_user).items(num_tweets):
            home_timeline_tweets.append(tweet)
        return home_timeline_tweets
```


```python
import re
from textblob import TextBlob
```


```python
import nltk
nltk.download('stopwords')
from nltk.corpus import stopwords
from nltk.stem.porter import PorterStemmer
```

    [nltk_data] Downloading package stopwords to
    [nltk_data]     C:\Users\Hemanth\AppData\Roaming\nltk_data...
    [nltk_data]   Package stopwords is already up-to-date!
    


```python
#### TWEET ANALYZER ####
class TweetAnalyzer():
    """
    Functionality for analyzing and categorizing content from tweets.
    """
    def clean_tweet(self, tweet):
        review = re.sub('[^a-zA-Z]', ' ', tweet)
        review = review.lower()
        review = review.split()
        ps = PorterStemmer()
        review = [ps.stem(word) for word in review if not word in set(stopwords.words('english'))]
        review = ' '.join(review)
        return review
    
    def analyze_sentiment(self, tweet):
        analysis = TextBlob(self.clean_tweet(tweet))
        
        if analysis.sentiment.polarity > 0:
            return 1
        elif analysis.sentiment.polarity == 0:
            return 0
        else:
            return -1
    
    def tweets_to_data_frame(self, tweets):
        df = pd.DataFrame(data=[tweet.text for tweet in tweets], columns=["tweets"])
        
        df['id'] = np.array([tweet.id for tweet in tweets])
        df['len'] = np.array([len(tweet.text) for tweet in tweets])
        df['date'] = np.array([tweet.created_at for tweet in tweets])
        df['source'] = np.array([tweet.source for tweet in tweets])
        df['likes'] = np.array([tweet.favorite_count for tweet in tweets])
        df['retweets'] = np.array([tweet.retweet_count for tweet in tweets])
        return df
```


```python
if __name__ == '__main__':
    twitter_client = TwitterClient()
    tweet_analyzer = TweetAnalyzer()
    
    api = twitter_client.get_twitter_client_api()
    
    tweets = api.user_timeline(screen_name='elonmusk', count=200)
    
    df = tweet_analyzer.tweets_to_data_frame(tweets)
    df['sentiment'] = np.array([tweet_analyzer.analyze_sentiment(tweet) for tweet in df['tweets']])
    
    print(df.head(20))
```

                                                   tweets                   id  \
    0   @cleantechnica Thanks for recognizing the grea...  1051591623069429760   
    1                        It is time to create a mecha  1051389235406598144   
    2                                    @SURUdenise NERV  1051381698682793985   
    3                 @shawnwasabi &lt;ahem&gt; Elon-chan  1051381006073118721   
    4   @StephanieeeeeJ Great. Also love Princess Mono...  1051379120322490368   
    5             Love Your Name\nhttps://t.co/fRU7nTWnML  1051377948916215810   
    6   @Sofiaan @vicentes @MrTommyCampbell @Tesla May...  1051367720111943681   
    7         @NikkoMan82 @fiquett @yousuck2020 Towelie 💗  1051360789561454593   
    8        @vicentes @MrTommyCampbell @Tesla Good idea!  1051230620234350592   
    9   @MrTommyCampbell @Tesla Whole beer keg fits in...  1051229599189807104   
    10            @ThingsWork Wow, mechanical logic gates  1051226218060632064   
    11  @ckharrison10 @yousuck2020 Comes with free tow...  1051222706371223552   
    12  @yousuck2020 Maybe wise to bring it just in ca...  1051219865661403136   
    13  @owillis About half my money is intended to he...  1050812486226599936   
    14  @owillis You should ask why I would want money...  1050811017221963776   
    15                                   @technosucks Yes  1050809457037328385   
    16  Tesla exists to help reduce risk of catastroph...  1050809258659344384   
    17  RT @RollingStone: One of the leading climate s...  1050798965371719681   
    18       Visual approximation https://t.co/sMn3Pv476Y  1050792830732394496   
    19   Teslaquila coming soon … https://t.co/AtoVGOtvVR  1050788043907448834   
    
        len                date              source   likes  retweets  sentiment  
    0    90 2018-10-14 21:52:35  Twitter for iPhone    9393       454          1  
    1    28 2018-10-14 08:28:22  Twitter for iPhone  183866     45014          0  
    2    16 2018-10-14 07:58:25  Twitter for iPhone   10216      2576          0  
    3    35 2018-10-14 07:55:40  Twitter for iPhone   49747     18134          0  
    4    51 2018-10-14 07:48:10  Twitter for iPhone    9681       954          1  
    5    38 2018-10-14 07:43:31  Twitter for iPhone   71203     15825          1  
    6   116 2018-10-14 07:02:52  Twitter for iPhone     775        30          1  
    7    43 2018-10-14 06:35:20  Twitter for iPhone     514        21          0  
    8    44 2018-10-13 21:58:05  Twitter for iPhone     793        22          1  
    9    61 2018-10-13 21:54:02  Twitter for iPhone   11755       720          1  
    10   39 2018-10-13 21:40:35  Twitter for iPhone    7895       381          1  
    11   94 2018-10-13 21:26:38  Twitter for iPhone    1426        64          1  
    12   53 2018-10-13 21:15:21  Twitter for iPhone   19091       816          1  
    13  143 2018-10-12 18:16:34  Twitter for iPhone    8242      1443         -1  
    14  139 2018-10-12 18:10:44  Twitter for iPhone   40683      3821          0  
    15   16 2018-10-12 18:04:32  Twitter for iPhone     557        25          0  
    16  140 2018-10-12 18:03:45  Twitter for iPhone  168957     27808          0  
    17  139 2018-10-12 17:22:50  Twitter for iPhone       0      1181          0  
    18   44 2018-10-12 16:58:28  Twitter for iPhone   51503      8859          0  
    19   48 2018-10-12 16:39:27  Twitter for iPhone   47141      6784          0  
    
