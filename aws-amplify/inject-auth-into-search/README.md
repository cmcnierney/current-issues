## Issue
The GraphQL Transform directive **@searchable** does not automatically inherit the **@auth** directive's restrictions. Unless handled, the default behavior would cause any user to receive Elasticsearch results from all users.

The issue is currently being tracked on [GitHub](https://github.com/aws-amplify/amplify-cli/issues/460)

### Existing Workaround

Override the response resolver to filter matching docs per @YikSanChan [GitHub](https://github.com/aws-amplify/amplify-js/issues/2113) 

### Alternative Workaround

Override the request resolver to inject the user context into the query.

For example, if your type is called `Example`, has the **@searchable** and **@auth** directives, and you've used the cli to generate your GraphQL resources, you'd copy `Query.searchExamples.req.vtl` from `<root>/amplify/backend/api/<appname>/build/resolvers` to `<root>/amplify/backend/api/<appname>/resolvers`.

#### 1. Fetch and store user context

To keep the code similar to the generated code, I'd copy this from one of the create* resolvers and place it at the top of your resolver.

```apache
#set( $identityValue = $util.defaultIfNull($ctx.identity.claims.get("username"), $util.defaultIfNull($ctx.identity.claims.get("cognito:username"), "___xamznone____")) )
```

#### 2. Inject user context into query per your project reqs

```apache
...
"query": {
  "bool": {
    "must": {
      "match_phrase": {
        "owner": "$identityValue"
      }
    },
    #if( $context.args.filter )
      "filter": $util.transform.toElasticsearchQueryDSL($ctx.args.filter)
    #else
      "filter": []
    #end
  }
}
...
```
Note the parentheses around the variable `"$identityValue"`.

#### 3. Connect the frontend

```jsx
const [search, setSearch] = useState("");
const getQueryWithSearchFilter = () => {
  let queryData =
    (search === "") 
      ? {}
      : {
      filter: {
        name: {
          matchPhrasePrefix: search
        }
      }
  };

  return graphqlOperation(queries.searchExamples, queryData);
};
...
<Connect
  query={getQueryWithSearchFilter()}
  subscription={graphqlOperation(subscriptions.onCreateExample)}
  onSubscriptionMsg={(prev, { onCreateExample }) => {
    prev.searchExamples.items.push(onCreateExample);
    return prev;
  }}
>
  {({ data: { searchExamples }, loading, error }) => {
    if (error) return <Notification error={error} />;
    if (loading || !searchExamples) return <Loader colorClass="has-background-primary" />;
    return renderList(searchExamples);
  }}
</Connect>
...

```

### Notes

* The resolver is assuming your **@auth** directive is basically just `@auth(rules: [{ allow: owner }])`. Anything more complex would need additional logic. The generated create* request resolvers are good starting points to look further.
* This solution allows a blank search term, effectively replacing the list* operations.
