---
layout: page
path: /postgraphile/make-wrap-resolvers-plugin/
title: makeWrapResolversPlugin
---

## makeWrapResolversPlugin

**NOTE: this documentation applies to PostGraphile v4.1.0+**

This plugin enables you to easily take actions before or after an existing GraphQL field resolver, or even to prevent the resolver being called.

IMPORTANT: Because PostGraphile uses the Graphile Engine look-ahead features, overriding a resolver may not affect the SQL that will be generated. If you want to influence how the system executes, only use this for modifying root-level resolvers (as these are responsible for generating the SQL and fetching from the database); however it's safe to use resolver wrapping for augmenting the returned values (for example masking private data) on any field.

```js
module.exports = makeWrapResolversPlugin({
  User: {
    async email(resolve, source, args, context, resolveInfo) {
      const result = await resolve();
      return result.toLowerCase();
    },
  },
});
```

`makeWrapResolversPlugin` takes either the resolver wrapper rules object directly, or a generator for this rules object, and returns a plugin.

```ts
function makeWrapResolversPlugin(
  rulesOrGenerator: ResolverWrapperRules | ResolverWrapperRulesGenerator
): Plugin;
```

The rules object is a two-level map of `typeName` (the name of a GraphQLObjectType) and `fieldName` (the name of one of the fields of this type) to either a rule for that field, or a resolver wrapper function for that field. The generator function accepts an `Options` object (which contains everything you may have added to `graphileBuildOptions` and more).

```ts
interface ResolverWrapperRules {
  [typeName: string]: {
    [fieldName: string]: ResolverWrapperRule | ResolverWrapperFn;
  };
}
type ResolverWrapperRulesGenerator = (options: Options) => ResolverWrapperRules;
```

A resolver wrapper function is similar to a GraphQL resolver, except it takes an additional argument (at the start) which allows delegating to the resolver that is being wrapped. If and when you call the `resolve` function, you may optionally pass one or more of the arguments `source, args, context, resolveInfo`; these will then override the values that the resolver will be passed. Calling `resolve()` with no arguments will just pass through the original values unmodified.

```ts
type ResolverWrapperFn = (
  resolve: GraphQLFieldResolver, // Delegates to the resolver we're wrapping
  source: TSource,
  args: TArgs,
  context: TContext,
  resolveInfo: GraphQLResolveInfo
) => any;
```

Should your wrapper require additional data, for example data about it's
sibling or child columns, then instead of specifying the wrapper directly you
can pass a rule object. The rule object should include the `resolve` method
(your wrapper) and can also include a list of requirements. It's advised that
your alias should begin with a dollar `$` symbol to prevent it conflicting
with any aliases generated by the system.

```ts
interface ResolverWrapperRequirements {
  childColumns?: Array<{ column: string; alias: string }>;
  siblingColumns?: Array<{ column: string; alias: string }>;
}
interface ResolverWrapperRule {
  requires?: ResolverWrapperRequirements;
  resolve?: ResolverWrapperFn;
  // subscribe?: ResolverWrapperFn;
}
```

For example, this plugin wraps the `User.email` field, returning `null` if
the user requesting the field is not the same as the user for which the email
was requested. (Note that the email is still retrieved from the database, it
is just not returned to the user.)

```js
makeWrapResolversPlugin({
  User: {
    email: {
      requires: {
        siblingColumns: [{ column: "id", alias: "$user_id" }],
      },
      resolve(resolver, user, args, context, _resolveInfo) {
        if (context.jwtClaims.user_id !== user.$user_id) return null;
        return resolver();
      },
    },
  },
});
```