---
title: Flask Web Development Notes
categories: 
- 技术笔记
- 读书笔记
tags: 
- read
date: 2018-8-9 22:54
updated: 2018-8-9 22:54
---

# Flask Web Development Notes

> 这是我暑假看 Miguel 的那本著名狗书时的笔记，书的版本是最新的 second edition。当时看的是原文版，以后会写一篇文章说阅读英语原文的经验



## Ch2 Basic Application Structure

### Run an app

```bash
export FLASK_APP=name.py
export FLASK_DEBUG=1
flask run
```

### Dynamic route

Flask supports the types `string`, `int`, `float`, and `path` for routes. The `path` type is a special `string` type.

### Response

*view function* could take as much as three arguments, *status code* and a *dictionary of headers*

```python
@app.route('/')
def index():
	return '<h1>Bad Request</h1>', 400
```



### Compatibility with older version

`app.run(debug=True)`

`app.add_url_rule()`: builds the map between URL and view function



```bash
flask run --host 0.0.0.0
```

```bash
git reset --hard
```

```bash
sudo apt-get install sqlite3 libsqlite3-dev
```



## Ch3 Templates

*business logic* and *presentation logic*, Jinja2 takes the work of *presentation*. More specifically, the application only sends variables to Jinja2, after that how the HTML page is presented is up to Jinja2.

### What this chapter's mainly about

Static web pages. Topics in this chapter:

- What are templates; how to inherit
- How to write URLs in Flask; `url_for()` helper function; relative & absolute URLs, the latter one less useful
- How to send static files (in template format)
- Miscellaneous: Bootstrap integration



## Ch4 Web Forms

### Form classes （technical)

`FlaskForm` defines the list of `field`s in each form, each represented by an object. Each `field` object have one or more *validators* attached. A *validator* is a function. Note the three terms here, *FlaskForm, field, validator*.

### Form submission with GET

Bad. GET doesn't have a *body*, so if submit with GET method, the *data* will be appended to the URL as a *query string*, and becomes visible in the browser's address bar.

Therefore, in *view function* that deals with forms, add a `POST` method like that:

```python
@app.route('/', methods=['GET', 'POST'])
```



### Miscellaneous

This method returns `True` when form is ~~submitted~~ submitted using POST method, and the data accepted by all *field validators*; used to *determine*, whether the form needs to be *rendered* or *processed*.

```python
if form.validate_on_submit():
	###
```

The first and only required argument to `url_for()` is the *endpoint* name.  By default, the endpoint of a route is the name of the view function attached to it. 

```python
return redirect(url_for('index'))
```



## Ch5 Databases

### Terms

column: attribute of the entity; row: actual data element that *assigns* values to columns.

table, record --> collection, document, from Relational to NoSQL

*primary key*: a special *column*; *foreign keys*: columns

### Sum

- With ORM middleman, one doesn't have to SQL semantics
- Apply different URLs in config could migrate from different (relational) databases
- Databases migration:
  - Framework: *Flask-Migrate*, the *Alembic wrapper*
  - Procedures: `flask db init`, creates a *migrations* directory (optional, only on a new project); `flask db migrate -m "message"`, *automatically* creates *migration script*, while `revision` manually; `flask db upgrade`
  - `flask db downgrade` and `... upgrade` gives db migrations log history like *git*



## Ch7 Large Application Structure

### Add custom commands via *click*

```python
@app.cli.command()
def thatWillBetheNameofCommand():
	"""This will be shown in the help messages"""
	pass
```

### Blueprint detail

- In chapter 7 about *blueprint*, the module where blueprint object is must import its routes' view functions, because

  > Importing these modules causes the routes and error handlers to be associated with the blueprint. 

- From Eg.7.7: the blueprint name before *endpoint* can be omitted, as `url_for('.index')`. *Redirects* within the same blueprint could use the shorter form, while crossing blueprints requires the fully qualified endpoint name that includes the blueprint name.





## Ch8 User Authentication

> Hashing functions are repeatable.

- **Q:** How could the `check_password_hash(hash, password)` check pwd without knowing the *salt* of it?

### WTForm: write your own validator with *validate_* prefix

- for `wtforms` *validate_* prefix defines custom validators; while in Python standard library `unittest` the *test_* prefix is used.
- any function referred the database, is defined in `model.py`, as methods in those ORM models. They access the database using `self.attribute`.

### Flask-Login detail

Flask-Login requires the application to designate a function to be invoked when the extension needs to load a user from the database given its identifier, registered with `@login_manager.user_loader` decorator.

`_get_user()` searches for a user ID in the *user session* --> invoke the function registered with the `user_loader` decorator, with the ID as argument



## Ch14 API

The process of converting an internal representation to a transport format such as JSON is called **serialization**.



## Todo

- [ ] decorators in Python
  They are functions that takes  a func as an arg then returns another arg

- [ ] **OOP** concepts: constructor, method&attribute
  (so far I *assume* that) Methods are funcs, attributes are data.

- [x] How blocks are really defined in bootstrap/base.html

- [ ] what `<>` means in Python code (`<{'key1': v, 'k2': v2}>`)

- [x] What is *label* in HTML
  makes the form text clickable; same style with it.

- [ ] > The remaining class variables are the attributes of the model, defined as instances of the `db.Column` class.

  what is *instance, class variable, attribute*

- [ ] `with` keyword in Python

- [x] `__name__`, `__file__` and other dunder variable (`basedir = os.path.abspath(os.path.dirname(__file__)) `)

  - `__name__` has two values, `"__main__"` (when being executed) or the module's name (when imported). With respect to Object, `__name__` is the special attribute of a class, function, method, descriptor, or generator instance. 
  - `__file__`, the absolute path of the module

- [ ] functionality of the `__init__.py` file in a directory

- [ ] **OOP:** `@property` decorator in a class

- [ ] **DB:** `users = db.relationship('User', backref='role', lazy='dynamic')`. "users" is a *column* of the table *role*. What does `lazy` kwarg mean

- [x] **DB:** `Role.query.filter_by(name='Administrator').first()`, type of the return value

  ```python
  User.query.filter_by(username='galt').first()  # output: <User 'john'>
  a = User.query.filter_by(username='galt').first()
  type(a)  # output: <class 'app.models.User'>
  ```

  On success, return value is the *row* in the table. `a.id` would return the `id` *column* of user `galt`.





---

- [ ] Linux command `export`
- [ ] `form.hidden_tag()` element for CSRF protection
- [x] How much freedom does ORM/ODM provide in choosing DB, only different *rational databases*?
  Yes.
- [ ] ```{{ wtf.quick_form(form) }}```, how will this template be rendered, search keyword in the book
  - `{{ name }}` construct references a **variable**, a **placeholder**
- [ ] How to initialize `SelectField`, 7.31
- [ ] `Post.query.order_by` will this query affect rows? 8.1



### After Flask:

- [ ] WTForm package *validator* source code, I assume it's just RegEx

- [ ] For example, the execution of the `send_async_email()` function can be sent to a [Celery](http://www.celeryproject.org/) task queue. (p.107)

- [ ] **Decorator:** How different orders affects the registered function

- [ ] **Thread:** *worker thread* in Flask doc

- [ ] **Iterator** and that: 

  ```python
  follows = [{'user': item.follower, 'timestamp': item.timestamp}
  for item in pagination.items]
  ```



Check yourself: 

- How to add a new row in a table, how to change a column of a row
- Page 182, how the foreign key of a table is added



---

- synopsis: brief summary

- delimiter (*programming*)

- miscellaneous

- *callable*, page 65

- cardinality alchemy, transparent (Ch5)

- > build a perfect **replica** of the virtual environment...



## Appendix

### Object in Python

type of methods:
- *instance method*, the 1st argument of it *must* be `self`; any method taking `self` as the 1st arg is a *instance method*
- *class method*, `@classmethod`, `cls` as the 1st arg. Calling: `Obj().count()`, where `Obj` is the name of a class
- *static method*， `@staticmethod`. E.g. `Obj().someStaticMethod()`

`super()` in Python2.x takes its own class as the first variable, like `super(Child, self).__init__(age, height)`. Also the *parent* shall be defined as `class Parent(object)`, having `object` as base class.

### REST API

- more business logic on client side
- client could *discover* resources using different URL compositions
- versioning for backward compatibility

### Compare to Mage Tutorial

1. _get
2. plain text and HTML email
3. from GitHub's *commit* history you could view specifically what changed in every commit

### Test & Caution

1. Think of **boundary values** while designing programs.

    - `current_user.confirmed`
    - `@login_required`
    - `self.email.lower()` ensure that the string is lowercase
    - `verify_password(email, password):` before loading user from DB to check pwd, `email` must be checked first not to be `''`

2. Crucial cases where proxies provided by Flask like`current_app` can't fake:

    - `current_app` can't fake its type as the actual object type
    - when sending *Signals* or passing data *to a background thread*

    best practice:

    ```python
    app = current_app._get_current_object()
    my_signal.send(app)
    ```

3. ```python
    from sqlalchemy.exc import IntegrityError
    try:
    	db.session.commit()
    except IntegrityError:
    	db.session.rollback()
    ```

4. `form.body.data = post.body`; how to render page according to attributes of the user;  Context processors make variables available to all templates during rendering

5. `from app.exceptions import`



### The power of Flask Shell

Page 214, static method `add_self_follows()` that manipulates existed *rows*