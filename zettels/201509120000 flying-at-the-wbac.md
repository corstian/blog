---
title: "Flying at the WBAC"
slug: "flying-at-the-wbac"
date: "2015-09-12"
summary: ""
references: 
  - '[[201712060000 mountain-gliding-2015]]'
  - '[[201705020000 cumulus-granitus-soaring-through-the-mountains]]'
---

#aviation

If there is one thing I hate about web development it is creating input forms. It's something I'm bad at, mostly because I do not have a cohesive mental model about how to deal with input props. In this post I'm about to explore some techniques to make form creation using React somewhat easier.


Part of my inspiration for this post comes from another blog post which you can find [here](https://www.digitalocean.com/community/tutorials/how-to-build-forms-in-react). In this post the author starts using reducers to abstract some of the logic of form handling away. It's something I can appreciate very much, and as such the goal of my little experiment will be to see how much I can golf a form in react. Throughout this post I'll work myself from mundane techniques to exciting shortcuts.

## Getting started with a simple form

Imagine I'm a react novice, and I'd like to create an input form. I'll build that something like this;

```jsx
export const FormElement = () => {
  const [name, setName] = useState()
  const [email, setEmail] = useState()

  return (
    <>
      <form onSubmit={(event) => {
         event.preventDefault()

         // Do something with the data here
        }}>
        <input value={name} onChange={(event) => setName(event.target.value)} />
        <input value={email} onChange={(event) => setEmail(event.target.value)} />
        <button type="submit">Submit</button>
      </form>
    </>
  )
}
```

There are multiple reasons why this code sucks:

1. The `useState` hook usage is directly proportional to the number of input fields
2. Each input field requires at least a `value` prop and `onChange` handler
3. When persisting the data, each of these fields will need to be called again

The result is that you'll be touching a single field multiple times before it is finally persisted. The possibility of forgetting just one is pretty big, with the result you'd need to go back to the editor to figure out which one of those calls you forgot. It's annoying at best. Demotivating at worst.

## Using a reducer instead of individual state
Thankfully it's possible to do away with those `useState` hooks by using a reducer. The concept of a reducer is not much different than that of a domain when talking about Domain Driven Design (DDD). There's a bag of data, and several operations which describe how one is allowed to change said data. We can restrict or open it up as much as we'd like, and even though that is not a requirement right now, we'd happily use the reducer to have all data at one place.

Our code starts to look something like this now;

```jsx
const initialState = {
  name: "",
  email: ""
}

const reducer = (state, action) => {
  return {
    ...state, // Make a copy of the existing state. We're not supposed to mutate variables directly.
    [action.name]: action.value
  }
}

export const FormElement = () => {
  const [state, setState] = useReducer(reducer, initialState)

  return (
    <>
      <form onSubmit={(event) => {
         event.preventDefault()

         // Do something with the data here
        }}>
        <input value={state.name} onChange={(event) => setState({ name: 'name', value: event.target.value})} />
        <input value={state.email} onChange={(event) => setState({ name: 'email', value: event.target.value})} />
        <button type="submit">Submit</button>
      </form>
    </>
  )
}
```

This is a slight improvement already. In the first place because the multiple `useState` hooks have been reduced to a reducer (pun intended). There is an even more important improvement we can make with this code. One that has to do with code reuse.

## Reusing code among input handlers

At this point it's fair game. We're trying to remove as much code as possible. So what we will keep is the reducer, because that thing handles our data. What we'll keep is the form, because that's out interface to the end user, and we'll keep some handlers, which are lifted out of the individual input elements;

```jsx
const reducer = (state, action) => {
  return {
    ...state,
    [action.name]: action.value
  }
}

const bindInput = (name, state, setState) => ({
  name,
  value: state[name],
  onChange: (event) => setState({ name, value: event.target.value })
})

export const FormElement = () => {
  const [state, setState] = useReducer(reducer, { /* Initial state is not required anymore */ })

  return (
    <>
      <form onSubmit={(event) => {
         event.preventDefault()

         // Do something with the data here
        }}>
        <input {...bindInput("name", state, setState)} />
        <input {...bindInput("email", state, setState)} />
        <button type="submit">Submit</button>
      </form>
    </>
  )
}
```

The most important aspect herein is the `bindInput` method. This takes an `name` argument, as well as accessors for the reducer. The output is an object which contains all the fields as required for a form input. The trick there is to deconstruct this object on the input itself such that they are bound to the input. The end result is a fairly monotonous form with a bit of fanfare around it. Minimizing the code block above we can get the following subtly different result;

```jsx
const reducer = (state, action) => {
  return {
    ...state,
    [action.name]: action.value
  }
}

const bindInput = (name, [state, setState]) => ({
  name,
  value: state[name],
  onChange: (event) => setState({ name, value: event.target.value })
})

export const FormElement = () => {
  const reducer = useReducer(reducer, { /* Initial state is not required anymore */ })

  return (
    <>
      <form onSubmit={(event) => {
         event.preventDefault()
         // Do something with the data here
        }}>
        
        <input {...bindInput("name", reducer)} />
        <input {...bindInput("email", reducer)} />

        <button type="submit">Submit</button>
      </form>
    </>
  )
}
```

## Iterating over input elements
At this point declarative form building is within reach. It'd be possible to create a structure describing your input form, outputting a ready to use form;

```jsx
const DeclarativeForm = () => {
  const form = {
    name: {
      type: text,
      regex: null,
      label: "First name",
      description: "We'd like to know your first name",
      placeholder: "John"
    }
  }

  return (
    <form>
      {
        Object
          .keys(form)
          .map(name => <input {...bindInput(name, reducer)} type={form[name].type} placeholder={form[name].placeholder} />)
      }
      <button type="submit">Submit</button>
    </form>
  )
}
```

## Combining this with a GraphQL API

Combine this with a great GraphQL API which provides metadata about the fields it requires, and it is literally possible to dynamically generate forms based on input types. This is something I will explore in the future, after the graphql-dotnet project supports custom type metadata. Though we're not there yet, it already has become easier to call mutations by bulk. This is done by giving the input elements the same names as the properties from an input type on the GraphQL api.

