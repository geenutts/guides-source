As they search for a rental, users might also want to narrow their search to a specific city. 
Let's build a component that will let them filter rentals by city.

To begin, let's generate our new component. 
We'll call this component `list-filter`, since all we want our component to do is filter the list of rentals based on input.

```bash
ember g component list-filter
```

As before, this creates a Handlebars template (`app/templates/components/list-filter.hbs`), 
a JavaScript file (`app/components/list-filter.js`), 
and a component integration test (`tests/integration/components/list-filter-test.js`).

Let's start with writing some tests to help us think through what we are doing.
The filter component should yield a list of filtered items to whatever is rendered inside of it, known as its inner template block.
We want our component to call out to 2 actions: one to provide a list of all items when no filter is provided and an action to search listings by city.

For our initial test, we'll simply check that all the cities we provide are rendered and that the listing object is accessible from the template.

Since we plan to use Ember Data as our model store, we need to make our action calls to fetch data asynchronous, so we'll return promises.
Because accessing persisted data is typically done asynchronously, we want to use the wait helper at the end of our test, which will wait for all promises to resolve before completing the test.

```javascript {data-filename=tests/integration/components/list-filter-test.js}
import Ember from 'ember';
import { moduleForComponent, test } from 'ember-qunit';
import hbs from 'htmlbars-inline-precompile';
import wait from 'ember-test-helpers/wait';

moduleForComponent('list-filter', 'Integration | Component | filter listing', {
  integration: true
});

const ITEMS = [{city: 'San Francisco'}, {city: 'Portland'}, {city: 'Seattle'}];
const FILTERED_ITEMS = [{city: 'San Francisco'}];

test('should initially load all listings', function (assert) {
  // we want our actions to return promises, since they are potentially fetching data asynchronously
  this.on('filterByCity', (val) => {
    if (val === '') {
      return Ember.RSVP.resolve(ITEMS);
    } else {
      return Ember.RSVP.resolve(FILTERED_ITEMS);
    }
  });
  
  // with an integration test, you can set up and use your component in the same way your application 
  // will use it.
  this.render(hbs`
    {{#list-filter filter=(action 'filterByCity') as |results|}}
      <ul>
      {{#each results as |item|}}
        <li class="city">
          {{item.city}}
        </li>
      {{/each}}
      </ul>
    {{/list-filter}}
  `);

  // the wait function will return a promise that will wait for all promises 
  // and xhr requests to resolve before running the contents of the then block.
  return wait().then(() => {
    assert.equal(this.$('.city').length, 3);
    assert.equal(this.$('.city').first().text().trim(), 'San Francisco');
  });
});
```
For our second test, we'll check that typing text in the filter will actually appropriately call the filter action and update the listings shown.

We force the action by generating a `keyUp` event on our input field, and then assert that only one item is rendered.

```javascript {data-filename=tests/integration/components/list-filter-test.js}
test('should update with matching listings', function (assert) {
  this.on('filterByCity', (val) => {
    if (val === '') {
      return Ember.RSVP.resolve(ITEMS);
    } else {
      return Ember.RSVP.resolve(FILTERED_ITEMS);
    }
  });

  this.render(hbs`
    {{#list-filter filter=(action 'filterByCity') as |results|}}
      <ul>
      {{#each results as |item|}}
        <li class="city">
          {{item.city}}
        </li>
      {{/each}}
      </ul>
    {{/list-filter}}
  `);
  
  // The keyup event here should invoke an action that will cause the list to be filtered
  this.$('.list-filter input').val('San').keyup();

  return wait().then(() => {
    assert.equal(this.$('.city').length, 1);
    assert.equal(this.$('.city').text().trim(), 'San Francisco');
  });
});

```

Next, in our `app/templates/index.hbs` file, we'll add our new `list-filter` component in a similar way to what we did in our test.  Instead of just showing the city, we'll use our `rental-listing` component to display details of the the rental.

```handlebars {data-filename=app/templates/index.hbs}
<div class="jumbo">
  <div class="right tomster"></div>
  <h2>Welcome!</h2>
  <p>
    We hope you find exactly what you're looking for in a place to stay.
    <br>Browse our listings, or use the search box above to narrow your search.
  </p>
  {{#link-to 'about' class="button"}}
    About Us
  {{/link-to}}
</div>

{{#list-filter
   filter=(action 'filterByCity')
   as |rentals|}}
  <ul class="results">
    {{#each rentals as |rentalUnit|}}
      <li>{{rental-listing rental=rentalUnit}}</li>
    {{/each}}
  </ul>
{{/list-filter}}
```

Now that we have failing tests and an idea of what we want our component contract to be, we'll implement the component.
We want the component to simply provide an input field and yield the results list to its block, so our template will be simple:

```handlebars {data-filename=app/templates/components/list-filter.hbs}
{{input value=value key-up=(action 'handleFilterEntry') class="light" placeholder="Filter By City"}}
{{yield results}}
```

The template contains an [`{{input}}`](../../templates/input-helpers/) helper that renders as a text field, in which the user can type a pattern to filter the list of cities used in a search. 
The `value` property of the `input` will be bound to the `value` property in our component.
The `key-up` property will be bound to the `handleFilterEntry` action.

Here is what the component's JavaScript looks like:

```javascript {data-filename=app/components/list-filter.js}
import Ember from 'ember';

export default Ember.Component.extend({
  classNames: ['list-filter'],
  value: '',

  init() {
    this._super(...arguments);
    this.get('filter')('').then((results) => this.set('results', results));
  },

  actions: {
    handleFilterEntry() {
      let filterInputValue = this.get('value');
      let filterAction = this.get('filter');
      filterAction(filterInputValue).then((filterResults) => this.set('results', filterResults));
    }
  }

});
```

We use the `init` hook to seed our initial listings by calling the `filter` action with an empty value.
Our `handleFilterEntry` action calls our filter action based on the `value` attribute set by our input helper.

Both the `filter` action is [passed](../../components/triggering-changes-with-actions/#toc_passing-the-action-to-the-component) in by the calling object. This is a pattern known as _closure actions_.

To implement these actions, we'll create the index controller for the application.  The index controller is executed when the user goes to the base (index) route for the application.

Generate a controller for the `index` page by running the following:

```bash
ember g controller index
```

Now, define your new controller like so:

```javascript {data-filename=app/controllers/index.js}
import Ember from 'ember';

export default Ember.Controller.extend({
  actions: {
    filterByCity(param) {
      if (param !== '') {
        return this.get('store').query('rental', { city: param });
      } else {
        return this.get('store').findAll('rental');
      }
    }
  }
});
```

When the user types in the text field in our component, the `filterByCity` action in the controller is called. 
This action takes in the `value` property, and filters the `rental` data for records in data store that match what the user has typed thus far. 
The result of the query is returned to the caller.

For this action to work, we need to replace our Mirage `config.js` file with the following, so that it can respond to our queries.

```javascript {data-filename=mirage/config.js}
export default function() {
  this.get('/rentals', function(db, request) {
    let rentals = [{
        type: 'rentals',
        id: 1,
        attributes: {
          title: 'Grand Old Mansion',
          owner: 'Veruca Salt',
          city: 'San Francisco',
          type: 'Estate',
          bedrooms: 15,
          image: 'https://upload.wikimedia.org/wikipedia/commons/c/cb/Crane_estate_(5).jpg'
        }
      }, {
        type: 'rentals',
        id: 2,
        attributes: {
          title: 'Urban Living',
          owner: 'Mike Teavee',
          city: 'Seattle',
          type: 'Condo',
          bedrooms: 1,
          image: 'https://upload.wikimedia.org/wikipedia/commons/2/20/Seattle_-_Barnes_and_Bell_Buildings.jpg'
        }
      }, {
        type: 'rentals',
        id: 3,
        attributes: {
          title: 'Downtown Charm',
          owner: 'Violet Beauregarde',
          city: 'Portland',
          type: 'Apartment',
          bedrooms: 3,
          image: 'https://upload.wikimedia.org/wikipedia/commons/f/f7/Wheeldon_Apartment_Building_-_Portland_Oregon.jpg'
        }
      }];

    if(request.queryParams.city !== undefined) {
      let filteredRentals = rentals.filter(function(i) {
        return i.attributes.city.toLowerCase().indexOf(request.queryParams.city.toLowerCase()) !== -1;
      });
      return { data: filteredRentals };
    } else {
      return { data: rentals };
    }
  });
}
```

After updating our mirage configuration, we should see passing tests, as well as a simple filter on your home screen, that will update the rental list as you type:

![home screen with filter component](/images/autocomplete-component/styled-super-rentals-filter.png)

![passing acceptance tests](/images/autocomplete-component/passing-acceptance-tests.png)
