# cheat_sheet

```python
auth = tweepy.OAuthHandler(CONSUMER_KEY, CONSUMER_SECRET)
auth.set_access_token(ACCESS_TOKEN, ACCESS_TOKEN_SECRET)

# Instantiate an instance of the API class from the tweepy library.
api = tweepy.API(auth, wait_on_rate_limit=True, wait_on_rate_limit_notify=True)


@shared_task(name='cleanup')
def cleanup():
    """
    Check database for records older than 7 days.
    Delete them if they exist.
    """
    Tweet.objects.filter(tweet_date__lte=datetime.now() - timedelta(days=7)).delete()


@shared_task(name='get_tweets')
def get_tweets():
    """Get some tweets from the twitter api and store them to the db."""

    # Subtasks
    chain = cleanup.s()
    chain()

    # Check for the minimum tweet_id and set it as max_id.
    # This ensures the API call doesn't keep getting the same tweets.
    max_id = min([tweet.tweet_id for tweet in Tweet.objects.all()])

    # Make the call to the Twitter Search API.
    tweets = api.search(
        q='#python',
        max_id=max_id,
        count=100
    )

    # Store the collected data into lists.
    tweets_date = [tweet.created_at for tweet in tweets]
    tweets_id = [tweet.id for tweet in tweets]
    tweets_source = [tweet.source for tweet in tweets]
    tweets_favorite_cnt = [tweet.favorite_count for tweet in tweets]
    tweets_retweet_cnt = [tweet.retweet_count for tweet in tweets]
    tweets_text = [tweet.text for tweet in tweets]

    # Iterate over these lists and save the items as fields for new records in the database.
    for i, j, k, l, m, n in zip(
            tweets_id,
            tweets_date,
            tweets_source,
            tweets_favorite_cnt,
            tweets_retweet_cnt,
            tweets_text
    ):
        try:
            Tweet.objects.create(
                tweet_id=i,
                tweet_date=j,
                tweet_source=k,
                tweet_favorite_cnt=l,
                tweet_retweet_cnt=m,
                tweet_text=n,
            )
        except IntegrityError:
            pass
  ```
