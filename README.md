# uncontrollable

Wrap a controlled react component, to allow spcific prop/handler pairs to be uncontrolled. Uncontrollable allows you to write pure react components, with minimal state, and then wrap them in a component that will manage state for prop/handlers if they are excluded.

## Install

```sh
npm i -S uncontrollable
```

### API

If you are a bit unsure on the _why_ of this module skip to the next section first.

require the module: `var uncontrollable = require('uncontrollable')'

#### `uncontrollable(Component, propHandlerHash, tapsHash)`

- `Component`: is a valid react component, such as the result of `createClass`
- `propHandlerHash`: define the pairs of prop/handlers you want to be uncontrollable eg. `{ value: 'onChange'}`
- `tapsHash`: sometimes you need to jump in the middle of a handler to adjust the value of another prop. You can specify a tap function to run before the handler like: `{ onToggle: () => this.setState({value: null })}`. this is generally not recommended but sometimes is necessary, this is an escape hatch to deal with interactions of multiple controlled/uncontrolled pairs in a component.

```js
    var uncontrollable =  require('uncontrollable');

    var UncontrolledCombobox = uncontrollable(
        Combobox, 
        { 
          value: 'onChange', 
          open: 'onToggle', 
          searchTerm: 'onSearch' //the current typed value (maybe it filters the dropdown list)
        },
        {
          'onToggle': function (){
            if ( this.props.searchTerm === undefined ) // if the consumer is not controlling searchTerm
              this.setState({ searchTerm: '' }) // reset the filter on close
          }
        })
```

### Use Case

One of the strengths of React is its extensibility module, enabled by a common mantra to push component state as high up the tree as possible. While great for enabling extermely flexible and easy to reason about components, this tends produce a lot of boilerplate to wire components up with every use. For simple components (like an input) this is usually a matter of tying the input `value` prop to a parent state property via its `onChange` handler

```jsx
  render: function() {
    return (
        <input type='text' 
          value={this.state.value} 
          onChange={ e => this.setState({ value: e.target.value })}/>
    )
  }
```
This pattern moves the responsibility of managing the `value` from the input to its parent and mimics "two-way" databinding. Sometimes, however, there is no need for the parent to manage the inputs state, all we need it "one-way" databinding in which case all we want to do is set the initial `value` of the input and let the input manage it from then on. React deals with this through "uncontrolled" inputs, where if you don't indicate taht you want to control the state of the input externally via a `value` prop it will just do the book keepying for you.

This is a great pattern, that we can make use of in our own Components. It is often best to build each component to be as stateless as possible, assuming that the parent will want to controll everything that makes sense. Take a simple Dropdown component

```js
var MyInput = React.createClass({

  propTypes: {
    value: React.PropTypes.string,
    onChange: React.PropTypes.func,
    open: React.PropTypes.bool,
    onToggle: React.PropTypes.func,
  },

  render: function() {
    return (
      <div>
        <input 
          value={this.props.value} 
          onChange={ e => this.props.onChange(e.value)}
        />
        <button onClick={ e => this.props.onToggle(!this.props.open)}>
          open
        </button>
        { this.props.open && 
          <ul className='open'>
            <li>option 1</li>
            <li>option 2</li>
          </ul>
        }
      </div>
    )
  }
});
```

Notice how we don't track any state in our simple dropdown? This is great because a consumer of our module will have the all the flexibility to decide waht the behaviour of the dropdown should be. Also notice our public API is (propTypes), it consists of a property we want set: `value`, `open` and a set of handlers that indicate _when_ we want them set: `onChange`, `onToggle`. It is up to the parent component to change the  `value` and `open` props in response to the handlers.

While this pattern offers a excellent amount of flexibility to our consumers it also requires them to write a bunch of boilerplate code that probably won't change much from use to use. In all likelihood they will always want to set `open` in response to `onToggle`, and only in rare cases, want to override that behavior. This is where our controlled/uncontrolled pattern comes in. 

We want to just handle the open/onToggle case ourselves if the consumer doesn't provide a `open` prop (indicating that they want to control it). Rather than complicating our dropdown component will all that logic that just gets in the way and obsures the business logic of our widget, we can add it later, by taking our dropdown and wrapping it inside another component that handles that for us.

Uncontrollable allows you serperate out the logic necessary to create controlled/uncontrolled inputs letting you focus on creating a completely controlled input and wrapping it later. This tends to be a lot simplier to reason about as well.

```js
    var uncontrollable =  require('uncontrollable');

    var UncontrollableMyInput = uncontrollable(
        SimpleDropdown, 
        /* define the pairs ->*/ 
        { value: 'onChange', open: 'onToggle' })

    <UncontrollableDropdown
      value={this.state.val} //we can still control these props if we want
      onChange={val => this.setState({ val })} 
      defaultopen={true} /> // or just let the UncontrollableDropdown handle it 
                            // and we just set an initial value (or leave it out completely)!
```

Now we don't need to worry about the open onToggle! the returned Component will jsut track that for us, and if we _do_ want to worry about it we can just provide `open` and `onToggle` props and the uncontrolled input will just pass them through.

The above is a contrived example but it allows you to wrap even more complex Components, giving you a lot of flexibiltity in the API you can offer a consumer of your Component. For every pair of prop/handlers you also get a defaultProp of the form "default[PropName]" so `value` -> `defaultValue`, and `open` -> `defaultOpen`, etc. [React Widgets](https://github.com/jquense/react-widgets) makes heavy use of this strategy, you can see it in action here: https://github.com/jquense/react-widgets/blob/master/src/Multiselect.jsx#L408
