---
id: ssr
title: Server-Side Rendering
sidebar_label: Server-Side Rendering (SSR)
slug: /nextjs/ssr
---

To enable SSR just set `ssr: true` in your `createTRPCNext` config callback.

:::info
When you enable SSR, tRPC will use `getInitialProps` to prefetch all queries on the server. This results in problems [like this](https://github.com/trpc/trpc/issues/596) when you use `getServerSideProps`, and solving it is out of our hands.

&nbsp;  
Alternatively, you can leave SSR disabled (the default) and use [Server-Side Helpers](server-side-helpers) to prefetch queries in `getStaticProps` or `getServerSideProps`.
:::

In order to execute queries properly during the server-side render step we need to add extra logic inside our `config`:

Additionally, consider [`Response Caching`](../server/caching.md).

```tsx title='utils/trpc.ts'
import { httpBatchLink } from '@trpc/client';
import { createTRPCNext } from '@trpc/next';
import superjson from 'superjson';
import type { AppRouter } from './api/trpc/[trpc]';

export const trpc = createTRPCNext<AppRouter>({
  config(opts) {
    const { ctx } = opts;
    if (typeof window !== 'undefined') {
      // during client requests
      return {
        transformer: superjson, // optional - adds superjson serialization
        links: [
          httpBatchLink({
            url: '/api/trpc',
          }),
        ],
      };
    }

    return {
      transformer: superjson, // optional - adds superjson serialization
      links: [
        httpBatchLink({
          // The server needs to know your app's full url
          url: `${getBaseUrl()}/api/trpc`,
          /**
           * Set custom request headers on every request from tRPC
           * @link https://trpc.io/docs/v10/header
           */
          headers() {
            if (!ctx?.req?.headers) {
              return {};
            }
            // To use SSR properly, you need to forward the client's headers to the server
            // This is so you can pass through things like cookies when we're server-side rendering

            const {
              // If you're using Node 18 before 18.15.0, omit the "connection" header
              connection: _connection,
              ...headers
            } = ctx.req.headers;
            return headers;
          },
        }),
      ],
    };
  },
  ssr: true,
});
```

```tsx title='pages/_app.tsx'
import type { AppProps } from 'next/app';
import React from 'react';
import { trpc } from '~/utils/trpc';

const MyApp: AppType = ({ Component, pageProps }: AppProps) => {
  return <Component {...pageProps} />;
};

export default trpc.withTRPC(MyApp);
```

## FAQ

### Q: Why do I need to forward the client's headers to the server manually? Why doesn't tRPC automatically do that for me?

While it's rare that you wouldn't want to forward the client's headers to the server when doing SSR, you might want to add things dynamically in the headers. Therefore, tRPC doesn't want to take responsibility for header keys colliding, etc.

### Q: Why do I need to delete the `connection` header when using SSR on Node 18?

If you don't remove the `connection` header, the data fetching will fail with `TRPCClientError: fetch failed` because `connection` is a [forbidden header name](https://developer.mozilla.org/en-US/docs/Glossary/Forbidden_header_name).

### Q: Why do I still see network requests being made in the Network tab?

By default, `@tanstack/react-query` (which we use for the data fetching hooks) refetches data on mount and window refocus, even if it's already got initial data via SSR. This ensures data is always up-to-date. See the page on [SSG](ssg) if you'd like to disable this behavior.
