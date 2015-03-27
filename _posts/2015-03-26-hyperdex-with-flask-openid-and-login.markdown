---
layout: post
title: HyperDex with Flask-OpenID and Login
category: posts
---

I've started work on building a cryptocurrency exchange with my friend [Akhil](http://akhilum.com), we're using [HyperDex][HD] and Python-Flask to power the project but more on that in a later post. This post is about using Flask-Login, Flask-OpenID and HyperDex. We couldn't find any guides to use OpenID (almost all examples used SQLite) the way we wanted it and I figured a little example could help someone else using this combination of technologies. Akhil was the one that implemented this, but I'm going to steal it all. >:)

This is the HyperDex space that you'll need to create. It stores the user's email address, nickname, and phone number. Of course you'll need the relevant WTForms and HTML pages to serve this.
{% highlight python %}
a.add_space('''
    space users
        key string email
        attributes
            string       name,
            int          phone
        subspace name
        subspace phone
''')
{% endhighlight %}

If your website is going to have users, it needs some way for them to sign up! I've removed the code I used to fetch requested data from the WTForm that is displayed on the HTML Page.
{% highlight python %}
oid = OpenID(app, os.path.join(basedir, 'tmp'))
@app.route('/signup', methods=['GET', 'POST'])
@oid.loginhandler
def signup():
    # Check if a user is already logged in.
    if g.user_email is not None:
        return redirect(url_for('index'))
    signup_form = SignupForm()
    # Validate user's signup form
    if signup_form.validate_on_submit():
        name   = request.form['name']
        openid = request.form['openid']
        # 3.
        session['login_type'] = "SIGNUP"
        session['name_var'] = name
        # 4.
        return oid.try_login(signup_form.openid.data, 
                            ask_for=['nickname', 'email'],
                            ask_for_optional=['fullname'])
    # 5.
    return render_template('signup.html', next=oid.get_next_url(), error=oid.fetch_error(),
                           title='f13x : Sign Up',
                           form=signup_form,
                           providers=app.config['OPENID_PROVIDERS'])
{% endhighlight %}

This is the login function, again, I've removed the code I used to fetch requested data from the WTForm.
{% highlight python %}
@app.route('/login', methods=['POST', 'GET'])
@oid.loginhandler
def login():
    # Check if a user is already logged in
    if g.user_email is not None:
        return redirect(oid.get_next_url())

    # Make a cookie called SIGNUP to pass to handler to let it know that
    # the validate was for SIGNUP and not a regular LOGIN
    session['remember_me'] = login_form.remember_me.data
    session['login_type'] = "LOGIN"
    # Validate the users email
    openid = request.form['openid']
    return oid.try_login(openid, ask_for=['email', 'nickname'],
                                     ask_for_optional=['fullname'])
    # Go back to the login form
    return render_template('login.html', 
                            next=oid.get_next_url(), 
                            error=oid.fetch_error(), 
                            title='f13x : Log In', 
                            providers=app.config['OPENID_PROVIDERS'])

{% endhighlight %}

You've probably noticed that we're storing the user's email address in the session variable. Flask stores encrypted session data in a cookie. Here's how to fetch that piece of information before each request.
{% highlight python %}
@app.before_request
def lookup_current_user():
    g.user_email = None
    if 'openid' in session:
        openid = str(session['openid'])
        if c.count('users', { 'email' : openid}) == 1:
            g.user_email = openid
{% endhighlight %}

The all important logout function
{% highlight python %}
@app.route('/logout')
@login_required
def logout():
    # logout a signed in user. Delete his cookie
    session.pop('openid', None)
    #flash(u'You were signed out')
    return redirect(oid.get_next_url())
{% endhighlight %}

You're definitely wondering by now where HyperDex fits in, well you need to store the user's emailId somewhere!
{% highlight python %}
@oid.after_login
def create_or_login(resp):
    # Check if the login was successful & emailId was provided by the OpenID provider
    if str(resp.email) == None or str(resp.email) == "":
        return redirect(url_for('login'))
    # Check if the user already exists.
    user_exists = False
    if c.count('users', { 'email' : str(resp.email) }) == 1L:
        user_exists = True
    # The user is trying to login
    if session['login_type'] == "LOGIN":
        # If this is an existing user
        if not user_exists:
            return redirect(url_for('signup'))
    # The user is trying to signup
    if session['login_type'] == "SIGNUP":
        # The user already exists, redirect him to the login page, we don't want him signing up again
        if user_exists:
            return redirect(url_for('login'))
        # Sign up was successful, initiate his data!
        c.put('users', str(resp.email), {'name' : str(resp.nickname), 'phone' : 1234567890})
    # Create a cookie with the user's data
    session['openid'] = str(resp.email)
    # Redirect the user to the page he was trying to fetch
    return redirect(request.args.get('next') or url_for('index'))
{% endhighlight %}

Let's say a user wants to update his phone number.
{% highlight python %}
@app.route('/update_phone', methods=['GET', 'POST'])
@login_required
def add_funds():
    if request.method == 'POST':
        new_phone_number = float(request.form['new_phone_number'])
        
        c.atomic_add('users', g.user_email, {'phone' : new_phone_number})
        
        # Redirect the user to the main page
        return redirect(url_for('index'))
{% endhighlight %}

The code might feel a bit out of place, I'll replace it with simpler code snippets if a lot of people don't understand it.

---

[HD]: http://hyperdex.org/
[twitter]: https://twitter.com/holman
