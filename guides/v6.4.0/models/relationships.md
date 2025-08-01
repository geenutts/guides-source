EmberData includes several built-in relationship types to help you
define how your models relate to each other.

### One-to-One

To declare a one-to-one relationship between two models, use
`belongsTo`:

```javascript {data-filename=app/models/user.js}
import Model, { belongsTo } from '@ember-data/model';

export default class UserModel extends Model {
  @belongsTo('profile') profile;
}
```

```javascript {data-filename=app/models/profile.js}
import Model, { belongsTo } from '@ember-data/model';

export default class ProfileModel extends Model {
  @belongsTo('user') user;
}
```

### One-to-Many

To declare a one-to-many relationship between two models, use
`belongsTo` in combination with `hasMany`, like this:

```javascript {data-filename=app/models/blog-post.js}
import Model, { hasMany } from '@ember-data/model';

export default class BlogPostModel extends Model {
  @hasMany('comment') comments;
}
```

```javascript {data-filename=app/models/comment.js}
import Model, { belongsTo } from '@ember-data/model';

export default class CommentModel extends Model {
  @belongsTo('blog-post') blogPost;
}
```

### Many-to-Many

To declare a many-to-many relationship between two models, use
`hasMany`:

```javascript {data-filename=app/models/blog-post.js}
import Model, { hasMany } from '@ember-data/model';

export default class BlogPostModel extends Model {
  @hasMany('tag') tags;
}
```

```javascript {data-filename=app/models/tag.js}
import Model, { hasMany } from '@ember-data/model';

export default class TagModel extends Model {
  @hasMany('blog-post') blogPosts;
}
```

### Explicit Inverses

EmberData will do its best to discover which relationships map to one
another. In the one-to-many code above, for example, EmberData can figure out that
changing the `comments` relationship should update the `blogPost`
relationship on the inverse because `blogPost` is the only relationship to
that model.

However, sometimes you may have multiple `belongsTo`/`hasMany`s for
the same type. You can specify which property on the related model is
the inverse using `belongsTo` or `hasMany`'s `inverse`
option. Relationships without an inverse can be indicated as such by
including `{ inverse: null }`.

```javascript {data-filename=app/models/comment.js}
import Model, { belongsTo } from '@ember-data/model';

export default class CommentModel extends Model {
  @belongsTo('blog-post', { inverse: null }) onePost;
  @belongsTo('blog-post') twoPost;
  @belongsTo('blog-post') redPost;
  @belongsTo('blog-post') bluePost;
}
```

```javascript {data-filename=app/models/blog-post.js}
import Model, { hasMany } from '@ember-data/model';

export default class BlogPostModel extends Model {
  @hasMany('comment', {
    inverse: 'redPost'
  })
  comments;
}
```

### Reflexive Relations

When you want to define a reflexive relation (a model that has a relationship to
itself), you must explicitly define the inverse relationship. If there
is no inverse relationship then you can set the inverse to `null`.

Here's an example of a one-to-many reflexive relationship:

```javascript {data-filename=app/models/folder.js}
import Model, { belongsTo, hasMany } from '@ember-data/model';

export default class FolderModel extends Model {
  @hasMany('folder', { inverse: 'parent' }) children;
  @belongsTo('folder', { inverse: 'children' }) parent;
}
```

Here's an example of a one-to-one reflexive relationship:

```javascript {data-filename=app/models/user.js}
import Model, { attr, belongsTo } from '@ember-data/model';

export default class UserModel extends Model {
  @attr('string') name;
  @belongsTo('user', { inverse: 'bestFriend' }) bestFriend;
}
```

You can also define a reflexive relationship that doesn't have an inverse:

```javascript {data-filename=app/models/folder.js}
import Model, { belongsTo } from '@ember-data/model';

export default class FolderModel extends Model {
  @belongsTo('folder', { inverse: null }) parent;
}
```

### Polymorphism

Polymorphism is a powerful concept which allows a developer
to abstract common functionality into a base class. Consider the
following example: a user with multiple payment methods. They
could have a linked PayPal account, and a couple credit cards on
file.

Note that, for polymorphism to work, EmberData expects a
"type" declaration polymorphic type via the reserved `type`
property on the model. Confused? See the API response below.

First, let's look at the model definitions:

```javascript {data-filename=app/models/user.js}
import Model, { hasMany } from '@ember-data/model';

export default class UserModel extends Model {
  @hasMany('payment-method', { polymorphic: true }) paymentMethods;
}
```

```javascript {data-filename=app/models/payment-method.js}
import Model, { belongsTo } from '@ember-data/model';

export default class PaymentMethodModel extends Model {
  @belongsTo('user', { inverse: 'paymentMethods' }) user;
}
```

```javascript {data-filename=app/models/payment-method-cc.js}
import { attr } from '@ember-data/model';
import PaymentMethod from './payment-method';

export default class PaymentMethodCcModel extends PaymentMethod {
  @attr last4;

  get obfuscatedIdentifier() {
    return `**** **** **** ${this.last4}`;
  }
}
```

```javascript {data-filename=app/models/payment-method-paypal.js}
import { attr } from '@ember-data/model';
import PaymentMethod from './payment-method'

export default class PaymentMethodPaypalModel extends PaymentMethod {
  @attr linkedEmail;

  get obfuscatedIdentifier() {
    let last5 = this.linkedEmail
      .split('')
      .reverse()
      .slice(0, 5)
      .reverse()
      .join('');

    return `••••${last5}`;
  }
}
```

And our API might setup these relationships like so:

```json
{
  "data": {
    "id": "8675309",
    "type": "user",
    "attributes": {
      "name": "Anfanie Farmeo"
    },
    "relationships": {
      "payment-methods": {
        "data": [
          {
            "id": "1",
            "type": "payment-method-paypal"
          },
          {
            "id": "2",
            "type": "payment-method-cc"
          },
          {
            "id": "3",
            "type": "payment-method-apple-pay"
          }
        ]
      }
    }
  },
  "included": [
    {
      "id": "1",
      "type": "payment-method-paypal",
      "attributes": {
        "linked-email": "ryan@gosling.io"
      }
    },
    {
      "id": "2",
      "type": "payment-method-cc",
      "attributes": {
        "last4": "1335"
      }
    },
    {
      "id": "3",
      "type": "payment-method-apple-pay",
      "attributes": {
        "last4": "5513"
      }
    }
  ]
}
```

### Readonly Nested Data

Some models may have properties that are deeply nested objects of
readonly data. The naïve solution would be to define models for each
nested object and use `hasMany` and `belongsTo` to recreate the nested
relationship. However, since readonly data will never need to be
updated and saved this often results in the creation of a great deal
of code for very little benefit. An alternate approach is to define
these relationships using an attribute with no transform
(`@attr`). This makes it easy to access readonly values in
other objects and templates without the overhead of defining
extraneous models.

### Creating Records

Let's assume that we have a `blog-post` and a `comment` model. A single blog post can have several comments linked to it. The correct relationship is shown below:

```javascript {data-filename=app/models/blog-post.js}
import Model, { hasMany } from '@ember-data/model';

export default class BlogPostModel extends Model {
  @hasMany('comment') comments;
}
```

```javascript {data-filename=app/models/comment.js}
import Model, { belongsTo } from '@ember-data/model';

export default class CommentModel extends Model {
  @belongsTo('blog-post') blogPost;
}
```

Now, suppose we want to add comments to an existing blogPost. We can do this in two ways, but for both of them, we first need to look up a blog post that is already loaded in the store, using its id:

```javascript
let myBlogPost = this.store.peekRecord('blog-post', 1);
```

Now we can either set the `belongsTo` relationship in our new comment, or, update the blogPost's `hasMany` relationship. As you might observe, we don't need to set both `hasMany` and `belongsTo` for a record. EmberData will do that for us.

First, let's look at setting the `belongsTo` relationship in our new comment:

```javascript
let comment = this.store.createRecord('comment', {
  blogPost: myBlogPost
});
comment.save();
```

In the above snippet, we have referenced `myBlogPost` while creating the record. This will let Ember know that the newly created comment belongs to `myBlogPost`.
This will create a new `comment` record and save it to the server. EmberData will also update `myBlogPost` to include our newly created comment in its `comments` relationship.

The second way of doing the same thing is to link the two records together by updating the blogPost's `hasMany` relationship as shown below:

```javascript
let comment = this.store.createRecord('comment', {});
let comments = await myBlogPost.comments;
comments.push(comment);
comment.save().then(function() {
  myBlogPost.save();
});
```

In this above case, the new comment's `belongsTo` relationship will be automatically set to the parent blogPost.

Although `createRecord` is fairly straightforward, the only thing to watch out for
is that you cannot assign a promise as a relationship, currently.

For example, if you want to set the `author` property of a blogPost, this would **not** work
if the `user` with id isn't already loaded into the store:

```javascript
this.store.createRecord('blog-post', {
  title: 'Rails is Omakase',
  body: 'Lorem ipsum',
  author: this.store.findRecord('user', 1)
});
```

However, you can easily set the relationship after the promise has fulfilled:

```javascript
let blogPost = this.store.createRecord('blog-post', {
  title: 'Rails is Omakase',
  body: 'Lorem ipsum'
});

this.store.findRecord('user', 1).then(function(user) {
  blogPost.author = user;
});
```

### Retrieving Related Records

When you request data from the server for a model that has relationships with one or more others,
you may want to retrieve records corresponding to those related models at the same time.
For example, when retrieving a blog post, you may need to access the comments associated
with the post as well.
The [JSON:API specification allows](http://jsonapi.org/format/#fetching-includes)
servers to accept a query parameter with the key `include` as a request to
include those related records in the response returned to the client.
The value of the parameter should be a comma-separated list of names of the
relationships required.

If you are using an adapter that supports JSON:API, such as Ember's default [`JSONAPIAdapter`](https://api.emberjs.com/ember-data/5.4.1/classes/JSONAPIAdapter),
you can easily add the `include` parameter to the server requests created by
the `findRecord()`, `findAll()`,
`query()` and `queryRecord()` methods.

`findRecord()` and `findAll()` each take an `options` argument in which you can
specify the `include` parameter.
For example, given a `post` model that has a `hasMany` relationship with a `comment` model,
when retrieving a specific post we can have the server also return that post's comments
as follows:

```javascript {data-filename=app/routes/post.js}
import Route from '@ember/routing/route';
import { service } from '@ember/service';

export default class PostRoute extends Route {
  @service store;
  model(params) {
    return this.store.findRecord('post', params.post_id, {
      include: 'comments'
    });
  }
}
```

The post's comments would then be available in your template as `model.comments`.

Nested relationships can be specified in the `include` parameter as a dot-separated sequence of relationship names.
So to request both the post's comments and the authors of those comments the request
would look like this:

```javascript {data-filename=app/routes/post.js}
import Route from '@ember/routing/route';
import { service } from '@ember/service';

export default class PostRoute extends Route {
  @service store;
  model(params) {
    return this.store.findRecord('post', params.post_id, {
      include: 'comments,comments.author'
    });
  }
}
```

The `query()` and `queryRecord()` methods each take a `query` argument that is
serialized directly into the URL query string and the `include` parameter may
form part of that argument.
For example:

```javascript {data-filename=app/routes/adele.js}
import Route from '@ember/routing/route';
import { service } from '@ember/service';

export default class AdeleRoute extends Route {
  @service store;
  model() {
    // GET to /artists?filter[name]=Adele&include=albums
    return this.store
      .query('artist', {
        filter: { name: 'Adele' },
        include: 'albums'
      })
      .then(function(artists) {
        return artists[0];
      });
  }
}
```

### Updating Existing Records

Sometimes we want to set relationships on already existing records. We can simply set a `belongsTo` relationship:

```javascript
let blogPost = this.store.peekRecord('blog-post', 1);
let comment = this.store.peekRecord('comment', 1);
comment.blogPost = blogPost;
comment.save();
```

Alternatively, we could update the `hasMany` relationship by pushing a record into the relationship:

```javascript
let blogPost = this.store.peekRecord('blog-post', 1);
let comment = this.store.peekRecord('comment', 1);
let comments = await blogPost.comments;
comments.push(comment);
blogPost.save();
```

### Removing Relationships

To remove a `belongsTo` relationship, we can set it to `null`, which will also remove it from the `hasMany` side:

```javascript
let comment = this.store.peekRecord('comment', 1);
comment.blogPost = null;
comment.save();
```

It is also possible to remove a record from a `hasMany` relationship:

```javascript
let blogPost = this.store.peekRecord('blog-post', 1);
let commentToRemove = this.store.peekRecord('comment', 1);
let comments = await blogPost.comments;
blogPost.comments = comments.filter((comment) => comment !== commentToRemove);
blogPost.save();
```

As in the earlier examples, the comment's `belongsTo` relationship will also be cleared by EmberData.

### Relationships as Promises

While working with relationships it is important to remember that they return promises.

For example, if we were to work on a blogPost's asynchronous comments, we would have to wait until the promise has fulfilled:

```javascript
let blogPost = this.store.peekRecord('blog-post', 1);

let comments = await blogPost.comments;
// now we can work with the comments
```

The same applies to `belongsTo` relationships:

```javascript
let comment = this.store.peekRecord('comment', 1);

let blogPost = await comment.blogPost;
// the blogPost is available here
```

Handlebars templates will automatically be updated to reflect a resolved promise. We can display a list of comments in a blogPost like so:

```handlebars
<ul>
  {{#each this.blogPost.comments as |comment|}}
    <li>{{comment.id}}</li>
  {{/each}}
</ul>
```

EmberData will query the server for the appropriate records and re-render the template once the data is received.
