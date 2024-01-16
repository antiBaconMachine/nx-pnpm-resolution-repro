# Nx node executor / pnpm workspaces(?) package resolution bug

This repro is designed to be a minimal repro of a bug with module resolution we found in our larger monorepo stack.

It consists of a typescript project which is dependent on a typescript library. The library is specified as a pnpm workspace dependency. The application is bundled with webpack though I believe this is actually irrelevant to the bug, none the less it is an accurate representation of our production setup.

The initial repo state is working we will introduce the bug below.

1. `nvm install`
2. `corepack enable`
3. `corepack prepare pnpm@8.12.1 --activate`
4. `pnpm i`
5. `npx nx run my-app:serve`

This will build both projects and output a console message.

The problem comes when the library is also a dependency of the top level package.json. This happens for instance if you are transitioning from using a single package.json to using pnpm workspaces with a package.json per project. Some projects still need the top level dep for a transitionary period.

1. `pnpm add -w @acme/hello-world`
2. `npx nx run my-app:serve`

This time the app will fail with `Error: Cannot find module '${REPO_ROOT}/dist/libs/hello-world`. We can see that this module path is not right for us as the library is configured to build in `${REPO_ROOT}/libs/hello-world/dist`.

## Partial explanation and workaround

I can explain _some_ of what is happening here.

The root cause is that nx tries to remap the module import location when the library is a buildable nx project. It does this using the `outputs` field on the library project. However when there is no top level dependency it seems that this is ignored and we just use the library from the node_modules directory in the application. I'm fuzzy on this detail would love some help fully explaining this part. 

If there is no output field set on the library project's build target then nx tries to do the remapping using the defaults listed in the [docs](https://nx.dev/reference/project-configuration#outputs)

```
    {workspaceRoot}/dist/{projectRoot},
    {projectRoot}/build,
    {projectRoot}/dist,
    {projectRoot}/public
```

We can see that number 3 is actually what we want, but unfortunately for us when it comes the path mapping that nx generates it arbitrarily picks the [first one](https://github.com/nrwl/nx/blob/42aefd87d0f971e1f3e773ff63bc3ff286351833/packages/js/src/executors/node/node.impl.ts#L324). We can see this causes the incorrect path above.

Note that this incorrect mapping is done regardless of whether there are one or two dependenices, but for some reason it only becomes a problem when the top level one exists.

## Workaround

We can set the outputs on the library build target to an empty array and this will avoid the issue due to [this line](https://github.com/nrwl/nx/blob/42aefd87d0f971e1f3e773ff63bc3ff286351833/packages/js/src/executors/node/node.impl.ts#L323). A better solution is probably to do as the [cache docs](https://nx.dev/concepts/how-caching-works#what-is-cached) suggest and use

```json
"targets": {
    "build": {
        "outputs": ["{options.outputPath}"],
        ...
    },
    ...
}
```

The `fix` branch demonstrates this mitigated setup. Don't forget to `pnpm install` after checking out.

## Questions

1. What is the purpose of this extra module mapping?
1. Why isn't the standard node require algo good enough for this case? It seems like we are trying to preferentially resolve the top level dep over the more specific one which is surprising behavior in node.
1. Why is the custom resolution ignored when there is no top level dependency?
1. Is this actually a bug or does it just need a documentation fix?

If someone can help me answer these questions then I'm happy to raise a PR to fix / mitigate.

Thanks.