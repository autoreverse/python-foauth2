# Overview

This is a fork of a fork of a fork. See (http://github.com/OfflineLabs/python-oauth2) for that history. A number of notable differences exist between this code and its forefathers:

* 0% unit test coverage.
* Lightweight at less than 200 lines, including blanks and docstrings.
* Completely removed all the OAuth 1.0 code.
* Completely removed all non-stdlib dependencies (goodbye httplib2, you won't be missed!).
* Implements a later version of the spec than it's parent.
* I'm not sure which version but google's auth2 API accepts this implementation.

# goo.gl URL shortener example

    client_id = 'xxxx.apps.googleusercontent.com'
    client_secret = 'xyzzy'
    auth_url = 'https://accounts.google.com/o/oauth2/auth'
    access_url = 'https://accounts.google.com/o/oauth2/token'
    scope = 'https://www.googleapis.com/auth/urlshortener'
    redirect_url = 'http://localhost/'

    import urlparse
    import foauth2
    client = foauth2.GooglAPI(client_id, client_secret)
    print "cut-n-paste the following URL to start the auth dance",
    print client.authorization_url()
    print "and copy the resulting link below"
    answer = raw_input()

    query_str = urlparse.urlparse(url_str).query
    params = urlparse.parse_qs(query_str, keep_blank_values=True)
    code = params['code'][0]
    access_token, refresh_token = client.redeem_token(code=code)

    # make a short url
    short_url = client.shorten('http://example.com')
    print short_url
    # get stats
    print client.stats(short_url)

GooglAPI inherits from the Client class.  You can use the client by itself but
then you have to pass in the URI and scope for the end points as arguments to
the authorize_url and redeem_code functions.  GooglAPI sets up the defaults:

    class GooglAPI(Client):
        user_agent = 'python-foauth2'
        # OAuth API
        auth_uri = 'https://accounts.google.com/o/oauth2/auth'
        refresh_uri = 'https://accounts.google.com/o/oauth2/token'
        scope = 'https://www.googleapis.com/auth/urlshortener'
        # Shortener API
        api_uri = 'https://www.googleapis.com/urlshortener/v1/url'

You are responsible for storing the access and refresh key for later use. Here is the full (and not very exciting) version of foauth.GooglAPI that we use.  It has a custom redirect URI, includes our goo.gl api key, and saves the access & refresh tokens to a django model names AuthKey. Define your own refresh_token() replacement to store stuff where you like.

    class GooglAPI(foauth2.GooglAPI):
        redirect_uri = 'http://hivefire.com/oauth'
        api_uri = 'https://www.googleapis.com/urlshortener/v1/url?key=%s' % GOOGL_KEY

        def refresh_token(self, *args, **kwargs):
            access, refresh = super(GooglAPI, self).refresh_token(*args, **kwargs)
            # save the updated access/refresh tokens
            query = AuthKey.objects.filter(service='goo.gl')
            query.update(access_token=access, access_secret=refresh,
                         acquired_date=datetime.datetime.now())
            return access, refresh
