## Thoughts on the PENDING Resource

Using `dependsOn` in simple cases like the one highlighted in the [README](https://github.com/SiftScience/resourcerer/blob/master/README.md) is pretty straightforward and very powerful. But `PENDING` resources bring additional complexities to your resource logic, some of which are enumerated here:


1. `PENDING` critical resources don’t contribute to `isLoading`/`hasErrored` states, but will keep your component from reaching a `hasLoaded` state. Semantically, this makes sense, because `this.props.hasLoaded` should only be true when all critical resources have loaded, regardless of when a resource’s request is made.
    
1. When a `PENDING` resource request is not in flight, its model prop will be an empty Model/Collection whose properties are frozen (the same empty model prop is passed when the resource has `ERRORED`). This is to more predictably handle our resources in our components. We don’t need to be defensive with syntax like:

     `this.props.todosCollection && this.props.todosCollection.toJSON()`. 
    
    1. When a resource is `LOADING`, its model prop will be an empty Model/Collection, but it will be the same instance of the model that will be populated when it is `LOADED` (this is an implementation detail that will likely never need to be considered in development).
        
1. When a previously-`PENDING` but currently `LOADED` resource has its dependent prop removed, it goes back to a `PENDING` state (recall that if the dependent prop is changed, it gets put back into a `LOADING` state while the new resource is fetched). This puts us in an interesting state:
    
    1. `hasInitiallyLoaded` will remain true, as expected. But the `PENDING` resource’s prop&mdash;assuming the resource’s `cacheFields` list includes the dependent prop&mdash;will now return to the empty model/collection, so any child component that may remain in place after `hasInitiallyLoaded` may need to keep that in mind (if `shouldComponentUpdate` returns false when the new resource fetches, then this shouldn’t matter&mdash;see the next point).
        
    2. Recall that we can provide the dependent prop in one of two ways:
        1. We can include it in another resource’s `provides` property, in which case the dependent prop gets saved as HOC state.
        2. We can modify the url in the component’s `componentWillReceiveProps`/`componentDidUpdate` (either url path or query parameter), which will filter the prop down.
    
        When we provide using method (a), the dependent prop can be changed but not removed. When we provide using method (b), the dependent prop can be changed or removed.
    
        So if we use method (a) and remove the dependent prop, we enter a state where `isLoading`, `hasLoaded`, and `hasErrored` are all false. And since we have to wait for a `componentWillReceiveProps`/`componentDidUpdate` to re-update the url with the dependent prop, a lifecycle passes with this state, and there’s really nothing we can do about it.
        
        And again—`hasInitiallyLoaded` is still true and the model prop is empty, which can cause layout issues if you use, for example, an overlaid loader over a previously-rendered component. For this reason, such a previously-rendered component should use `nextProps.hasLoaded` instead of `!nextProps.isLoading` in its `shouldComponentUpdate`:

    ```js
      // overlay-wrapped component, where a loader will show over previously-rendered children,
      // which we want to then not update. but this component is also a child of a `withResources`
      // component that has a dependent resource
      shouldComponentUpdate(nextProps) {
        // using `return !nextProps.isLoading;` would update the component in the above
        // scenario, even though `nextProps.myDependentModel` would be empty
        return nextProps.hasLoaded;
      }
    
      // this assumes that the parent is handling the `hasErrored` state. if it is not, then
      // you may need to instead use:
      shouldComponentUpdate(nextProps) {
        return !(nextProps.isLoading || isPending(nextProps.myDependentLoadingState));
      }
    ```

    As an example of how we might be able to remove a dependent prop, consider someone navigating to a `/todos` url that auto-navigates to the first todo item and displays its details. The `todoItem` details resource depends on a `todoId` prop, which it gets in a `componentWillReceiveProps` via changing the url once the `todos` resource loads. So now we’re at `/todos/todo1234`. But if the user clicks the back button, we’ll be back at `/todos` with a cached `todos` resource and `PENDING` `todoItem` resource, and all three loading states set to `false`.
       
## Implicit dependent resources

Another way to effectively have a dependent resource is to use a conditional in your `getResources` method:
    
```js
@withResources((props, ResourceKeys) => ({
  [ResourceKeys.TODOS]: {},
  ...(props.todoId ? {[ResourceKeys.TODO_ITEM]: {attributes: {id: props.todoId}} : {})
}))
```

In general, using `dependsOn` is much more preferable, both in terms of semantics and functionality. The key difference here is that the dependent resource does not get put into a `PENDING` state, and `hasLoaded` depends on an unpredictable number of resources&mdash;for example, in the above scenario, what happens if `props.todoId` never arrives? Using `dependsOn`, `hasLoaded` would not be true, but using the conditional, it would be. This means that with the conditional, you can’t freely make assumptions behind the `this.props.hasLoaded`  flag:

```jsx
{this.props.hasLoaded ? (
  // with the conditional, you don't know which resources are available. with
  // `dependsOn`, you do
) : null}
```

That doesn’t mean that the conditional can’t be useful&mdash;it’s just that its use should be relegated to components that have two discrete forms&mdash;one in which the dependent prop is always present, and one in which the dependent prop is never present. If you’re unsure whether a prop might exist, notably because it comes from a providing resource, you should use `dependsOn`. A good example of when to use a conditional is in this fake component that sometimes fetches a user model and sometimes fetches an order model depending on the presence of an `orderId` prop (user resource config not shown):

```js
@withResources(({userId, orderId}, ResourceKeys) => ({
  ...!userId && orderId ? {
    [ResourceKeys.ORDER]: {
      noncritical: true,
      attributes: {id: orderId}
    }
  } : {}
}))
```

In this case, when the component is used as an order component (denoted by the presence of the `orderId` prop), we fetch the `order` resource. Otherwise, we don’t.

For all other uses of dependent resources, we should use `dependsOn`.


## Unfetched Resources

You may find, at some point in your application, that you have a `PUT` endpoint for a resource but no `GET`; the 'read portion' of the resource is received as part of some parent resource. For example, imagine you have an accounts resource at `/accounts/{account_id}`, whose response has a `config` property with some account configuration settings. To update the configuration, you make a `PUT` to `/accounts/{account_id}/config`. But reading comes from the parent resource. `resourcerer` supports this via a `providesModels` static property on the model:

```js
// resources_conifg.js
import {ResourceKeys, UnfetchedResources} from 'resourcerer/config';

ResourceKeys.add({
  ACCOUNT: 'account',
  ACCOUNT_CONFIG: 'accountConfig'
});

// add the ACCOUNT_CONFIG key to our set of UnfetchedResources
UnfetchedResources.add(ResourceKeys.ACCOUNT_CONFIG);




// account_model.js
export default Backbone.Model.extend({
  initialize(attrs, options) {
    this.accountId = options.accountId;
  },
  
  url: () => `/accounts/${this.accountId}`
}, {
  cacheKeys: ['accountId'],
  providesModels: (accountModel, ResourceKeys) => [{
    attributes: accountModel.get('config'),
    modelKey: ResourceKeys.ACCOUNT_CONFIG,
    options: {accountModel}
  }]
})
```

The `providesModels` property is a function that takes the parent model and the `ResourceKeys` as arguments and returns an array of resource configs. The resource configs have the same schema as the those used in our general `withResources` executor functions. What this tells `resourcerer` to do is, after the parent model returns, instantiate the child model(s) and place them into the `ModelCache`. In other components, you can then access the model you know to exist in a `withResources` declaration:

```js
// child_component.jsx, rendered only after the account model is known to return
@withResources((props}, ResourceKeys) => ({
  [ResourceKeys.ACCOUNT_CONFIG]: {}
}))
class ChildComponent extends React.Component {
  // component has this.props.accountConfigModel from the cache!
  onClickSomething() {
    // and now you can update the config directly :)
    this.props.accountConfigModel.save();
  }
}
```

In general, this should be used in cases where you can ascertain that the parent model has returned before trying to access the child model. However, if by chance it has not, and the child is not found in the cache, `resourcerer` will still not attempt to fetch it, because it is listed within the `UnfetchedResources` set. In that case, the model will get instantiated with no seed attributes and passed as a prop.

Also, note that the `modelKey` property is required here instead of optionally being inferred from the resource config's object property, as is the case in our general `withResources` declarations. This is because here, in contrast, the models are simply placed in the cache and not actually used as props for any component, so they don't need to be named. Accordingly, resource configs are also returned as a list here instead of an object.

The resource config objects within `providesModels` have the same schema, as mentioned, as normal. But they also accept an additional optional property, `shouldCache`, which is a function that takes the parent model and the resource config as an argument. If the function exists and returns false, the model will not get instantiated nor placed in the cache:

```js
// account_model.js
export default Backbone.Model.extend({
  // ...
}, {
  cacheKeys: ['accountId'],
  providesModels: (accountModel, ResourceKeys) => [{
    attributes: accountModel.get('config'),
    modelKey: ResourceKeys.ACCOUNT_CONFIG,
    options: {accountModel},
    // if this returns false, account config won't get instantiated and placed in the ModelCache
    shouldCache: (accountModel, config) => accountModel.get('state') === 'ACTIVE'
  }]
})
```
