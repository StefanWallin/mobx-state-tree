I recently joined the core team of maintainers for [MobX-State-Tree](https://mobx-state-tree.js.org/intro/welcome), and I've been digging into the internals to learn about how it all works under the hood.

This blog post is the result of some experimentation, and my intention is to use it as partial documentation, and to share context for a pull request adding test coverage to the library. Unless you're deeply interested in the internals of MobX-State-Tree, you may find it a little dry.

But if you're excited about state management or open source JavaScript, you've come to the right place! Let's get to it. I'll be taking a look at some of the code in the commit with hash [048f1ebf0d2c49983ec8914d78162882bc1a3e3e](https://github.com/mobxjs/mobx-state-tree/tree/048f1ebf0d2c49983ec8914d78162882bc1a3e3e/packages/mobx-state-tree)

## Introduction

One of the most common things you'll do in MobX-State-Tree is define models, something like this:

```js
import { types } from "mobx-state-tree"

// A tweet has a body (which is text) and whether it's read or not
const Tweet = types
    .model("Tweet", {
        body: types.string,
        read: false // automatically inferred as type "boolean" with default "false"
    })
    .actions((tweet) => ({
        toggle() {
            tweet.read = !tweet.read
        }
    }))
```

I think the API is pretty ergonomic and straightforward. But I wanted to get a sense of what we really expect as input, what comes back, and how MST figures it all out.

## Input

Turns out, `types.model` is pretty flexible about what it'll accept. [The function definition](https://github.com/mobxjs/mobx-state-tree/blob/048f1ebf0d2c49983ec8914d78162882bc1a3e3e/packages/mobx-state-tree/src/types/complex-types/model.ts#L726) uses [TypeScript function overloads](https://www.typescriptlang.org/docs/handbook/2/functions.html#function-overloads) to handle three different types of input:

In the most specific case, we expect to get a `name` as a string and set of `properties` (technically typed as a [`ModelPropertiesDeclaration`](https://github.com/mobxjs/mobx-state-tree/blob/048f1ebf0d2c49983ec8914d78162882bc1a3e3e/packages/mobx-state-tree/src/types/complex-types/model.ts#L73)), but it's probably easiest to think of it as a spicy `Object`.

Less specifically, the function can be invoked without the `name` argument, but with a set of properties (again, `ModelPropertiesDeclaration`).

Finally, in the most general case, `types.model` will accept pretty much... any arguments at all. [The catch-all, actual implementation of the function](https://github.com/mobxjs/mobx-state-tree/blob/048f1ebf0d2c49983ec8914d78162882bc1a3e3e/packages/mobx-state-tree/src/types/complex-types/model.ts#L738) looks like this:

```ts
export function model(...args: any[]): any {
    const name = typeof args[0] === "string" ? args.shift() : "AnonymousModel"
    const properties = args.shift() || {}
    return new ModelType({ name, properties })
}
```

Here's what we do when we invoke `types.model`:

1. Figure out the name for the model (either `args[0]` or `AnonymousModel`)
2. Determine out the properties to give to the model (`args[0]` if no name given, `args[1]` if a name is given, and then finally an empty object, `{}` in all other cases)
3. Provide these arguments to the `ModelType` constructor.

## Output

Once we pass this information to the `ModelType` class, we'll get back some instance that will satisfy the [`IModelType` interface](https://github.com/mobxjs/mobx-state-tree/blob/048f1ebf0d2c49983ec8914d78162882bc1a3e3e/packages/mobx-state-tree/src/types/complex-types/model.ts#L177).

There's lots to be said about the `IModelType` interface, so I'll save that for a future exploration. At a high level, this interface is pretty much what describes the API surface of the [`types.model` itself](https://mobx-state-tree.js.org/API/#model). We need it to have:

1. A set of properties that describe the tree and its possible state.
2. A function called `views`, which will allow us to set up and apply [views](https://mobx-state-tree.js.org//intro/getting-started#model-views) to the instance.
3. A function called `actions`, which will allow us to set up [actions](https://mobx-state-tree.js.org/concepts/actions) that can modify the instance and its subtree(s)
4. A function called `volatile`, which will allow us to set up [volatile](https://mobx-state-tree.js.org/concepts/volatiles) state for the object.
5. An `extend` method, which allows us to share state between views and actions.
6. Methods called `preProcessSnapshot` and `postProcessSnapshot`, which can transform snapshots or take action after processing snapshots in the instance.

## How we get there

For the scope of this post, here's what we can expect to happen in the [constructor](https://github.com/mobxjs/mobx-state-tree/blob/048f1ebf0d2c49983ec8914d78162882bc1a3e3e/packages/mobx-state-tree/src/types/complex-types/model.ts#L342):

1. We set the name of the model
2. We parse the provided properties into an actual instance of [`ModelProperties`](https://github.com/mobxjs/mobx-state-tree/blob/048f1ebf0d2c49983ec8914d78162882bc1a3e3e/packages/mobx-state-tree/src/types/complex-types/model.ts#L65) (notice how we took an `ModelPropertiesDeclaration` and turned it into an actual `ModelProperties`)
3. We freeze the properties
4. We determine the identifier attribute if there is one.

Here's what that looks like:

```ts
 constructor(opts: ModelTypeConfig) {
        super(opts.name || defaultObjectOptions.name)
        Object.assign(this, defaultObjectOptions, opts)
        // ensures that any default value gets converted to its related type
        this.properties = toPropertiesObject(this.properties) as PROPS
        freeze(this.properties) // make sure nobody messes with it
        this.propertyNames = Object.keys(this.properties)
        this.identifierAttribute = this._getIdentifierAttribute()
    }
```

## Writing some tests

I hope you're feeling pretty good about the high level overview so far. This helped me to map out what's going on with my models in MobX-State-Tree. Let's see it in action! I want to write some tests around the behavior I've described. **TODO:** finish this with the actual tests after writing.

### Testing names

1. Providing a string as the first argument should set it as the model's name.
2. Providing an empty string as the first argument should set it as the model's name.
3. Providing a non-string argument as the first argument should set the model's name as 'AnonymousModel'.

### Testing properties

1. Providing a string as the first argument and an object as the second argument should use the object's properties in the model.
1. Providing an object as the first argument should parse and use its properties.
1. Providing a string as the first argument and a falsy value as the second argument should result in an empty set of properties

### Identifiers

1. If no identifier attribute is provided, the `identifierAttribute` should be `undefined`.
1. If an identifier attribute is provided, the identifierAttribute should be set for the object.
1. If an identifier attribute has already been provided, an error should be thrown when attempting to provide a second one

### Edge cases

1. When we provide no arguments to the function, the model will be named `AnonymousModel`
1. When we provide no arguments to the fucntion, the model will have no properties.
1. When we provide an invalid name value, but a valid property object, the model will be named `AnonymousModel`.
1. When we provide an invalid name value, but a valid property object, the model will have no properties (this is maybe a sneaky error if someone is not paying attention closely at author-time).
1. When we provide three arguments to the function, the model gets the correct name.
1. When we provide three arguments to the function, the model gets the correct properties.

## Stuff I learned writing this post

Here's what really jumped out at me as I did the code archaeology for this post:

1. TypeScript function overloading is cool, but it's confusing to read if you don't already have experience with it. We should consider documenting it more clearly where we use it.
2. I'm guessing there are a lot of advance TypeScript features we use in MobX-State-Tree that warrant discussion and documentation for newcomers (or even just ourselves in the future).
3. I really like the transformation of `ModelPropertiesDeclaration` -> `ModelProperties`. The TypeScript naming here communicates the intent quite clearly.
4. It's interesting that we literally `shift` the args in the `model` function. I wonder if there's any particular advantage or disadvantage to modifying the input directly like that.
5. `toPropertiesObject` is pretty complex - I'll want to cover that in its own investigation.
6. I think in my head, I've always thought of `actions`, `views`, `volatile`, and so forth as "properties" of the MobX-State-Tree instance, rather than methods that we call on the instance itself. From a quick glance, it looks like those methods do in fact transform the instance and return an augmented version of it, but there is a fundamental difference between the properties object passed in to the constructor, and the transformed instance after calling those methods. When people write MST, I would bet most of them also think of actions and views as a "part" of the input to the model definition - rather than a separate modification to the underlying instance.
