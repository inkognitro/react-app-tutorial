[« previous](06-form.md) | [next »](08-apiv1.md)

## 7. Collections
In this chapter we are going to create a provider for entity arrays which
can be passed as prop (not only) to table or dropdown components.
I am sure we can work out a way to standardize the process for fetching entities,
independent of where they are stored and provided:
Either from an API endpoint or just from a simple array.

### 7.1 Understand the principles
First, I'd like to introduce some terms of the Java world in relation to an array of objects:

> The Collection in Java is a framework that provides an architecture to store and manipulate the group of objects.
> Java Collections can achieve all the operations that you perform on a data such as searching, sorting, insertion,
> manipulation, and deletion.
> 
> `https://www.javatpoint.com/collections-in-java, 2022-06-21`

As of this, I think it's a better idea to call the data itself - without any functionality - the "collection".
The processor of operations like fetching, creating etc. could be called "collection manager"
or in case of just providing collections, the "collection provider".
Java does not make a difference about these two.
This aligns also with the naming in the React-Admin framework.

> :bulb: Marmelab's React-Admin uses a similar pattern called
> [data providers](https://marmelab.com/react-admin/DataProviders.html).
> Even creation, updating and deletion of one or multiple entities is supported.
>
> `https://www.javatpoint.com/collections-in-java, 2022-06-21`

I suggest only supporting the fetch of entities, with filter, sorting and pagination params
by such a "collection provider". As of my experience in frontend programming, writing and storing data is often an 
individually designed process. Because of this we'll not support write operations in our collection providers,
or at least for now.

### 7.2 Define the interface
Let's try to create an interface for the collection provider which takes a query holding
the information about filtering, sorting and pagination params.

```typescript
//src/packages/core/collection/collection.ts

export type Sorting = {
    field: string;
    direction: 'asc' | 'desc';
}[];

export type Filters = {
    [field: string]: string | number | boolean;
};

// Let's wrap the data in an "entry" holding a recognizable "key", because
// not every entity should be forced to hold a primary key called "id".
// Therefore we need to be able to create a custom and recognizable key, directly after
// a fetch process has been finished. 
export type Entry<Data = any> = {
    key: string;
    data: Data;
};

// Furthermore we should support pagination by a "limit" and "offset" param.
// This keeps the door open to reload some entries for every constellation.
export type CollectionQuery = {
    search?: string;
    offset: number;
    limit: number;
    filters: Filters;
    sorting: Sorting;
};

// Let's provide a reusable default query
export const defaultQuery: CollectionQuery = {
    offset: 0,
    limit: 10,
    filters: {},
    sorting: [],
};

// Furthermore we should provide a collection information object holding at least all params
// of the query itself.
// With this we store the information for which filters, sorting and pagination params the delivered entries
// were received. This should normally correspond with the params of the query, but HAS NOT TO!
export type CollectionInfo = CollectionQuery & {
    totalCount: number;
    filteredCount: number;
};

export function createQuery(provider: CollectionProvider): CollectionQuery {
    const latestQueryInfo = provider.latestQueryInfo ?? {};
    return {
        ...defaultQuery,
        ...latestQueryInfo,
    };
}

// To display useful information we should support things like "isFetching" in the
// state of a collection provider. With this we e.g. could show a loader icon or so.
export type CollectionProviderState<D = any> = {
    key: string;
    isFetching: boolean;
    hasInitialFetchBeenDone: boolean;
    entries: Entry<D>[];
    latestQueryInfo?: CollectionInfo;
};

export type EntriesOperation = 'append' | 'replace';

export type CollectionProvider<D = any> = CollectionProviderState<D> & {
    fetch: (query?: CollectionQuery, op?: EntriesOperation) => Promise<any>;
};
```

So now that we have an interface defined for a collection provider, let's create an implementation in the next step.

### 7.3 Create the `ArrayCollectionProvider` implementation
It is always useful to be able to provide entries from a simple array.
But we don't know the format of an array entity yet.
We only know it in the place where we want to implement the provider:
In the component who is using the provider.
So let's create a hook which does create us the desired provider from an array:

```typescript jsx
// src/packages/core/collection/arrayProvider.tsx

import { useEffect, useRef, useState } from 'react';
import {
    CollectionProvider,
    CollectionProviderState,
    CollectionQuery,
    defaultQuery,
    EntriesOperation,
    Entry,
} from './collection';
import { v4 } from 'uuid';

function getPaginatedEntries(entries: Entry[], query: CollectionQuery) {
    return entries.slice(query.offset, query.offset + query.limit);
}

type EntriesToShowConfig = {
    currentEntries: Entry[];
    availableEntries: Entry[];
    query?: CollectionQuery;
    op?: EntriesOperation;
};

function createEntriesToShow(props: EntriesToShowConfig): Entry[] {
    const query = props.query ?? defaultQuery;
    let entriesToShow = props.availableEntries;
    // todo: support filtering (not part of the tutorial yet)
    // todo: support sorting (not part of the tutorial yet)
    entriesToShow = getPaginatedEntries(entriesToShow, query);
    entriesToShow =
        props.op === 'append' && props.currentEntries ? [...props.currentEntries, ...entriesToShow] : entriesToShow;
    return entriesToShow;
}

export type ArrayCollectionProviderProps<D = any> = {
    dataArray: D[];
    createEntryKey: (data: D) => string;
};

export function useArrayCollectionProvider<D = any>(props: ArrayCollectionProviderProps<D>): CollectionProvider<D> {
    const availableEntriesRef = useRef<Entry<D>[]>(
        props.dataArray.map((data) => ({
            key: props.createEntryKey(data),
            data,
        }))
    );
    const [state, setState] = useState<CollectionProviderState>({
        key: v4(),
        isFetching: false,
        entries: [],
        hasInitialFetchBeenDone: false,
    });
    useEffect(() => {
        availableEntriesRef.current = props.dataArray.map((data) => ({
            key: props.createEntryKey(data),
            data,
        }));
        setState({
            ...state,
            entries: createEntriesToShow({
                availableEntries: availableEntriesRef.current,
                query: state.latestQueryInfo,
                currentEntries: state.entries,
            }),
            hasInitialFetchBeenDone: true,
        });
    }, [JSON.stringify(props.dataArray), setState, availableEntriesRef]);
    function fetch(query: CollectionQuery = defaultQuery, op: EntriesOperation = 'replace') {
        return new Promise<void>((resolve) => {
            resolve();
            setState({
                ...state,
                entries: createEntriesToShow({
                    availableEntries: availableEntriesRef.current,
                    query: query,
                    currentEntries: state.entries,
                    op,
                }),
                hasInitialFetchBeenDone: true,
            });
        });
    }
    return { ...state, fetch };
}
```

On the other hand, if we'd like to have a provider which should provide a collection from an API endpoint,
we need to write another implementation for this, which is called `ApiCollectionProvider` or so.
As for now, we have no api package yet, so let's continue with the `ArrayCollectionProvider` in the next steps.

Don't forget to export everything in an `index.ts`:

```typescript
// src/packages/core/collection/index.ts

export * from './collection';
export * from './arrayProvider';
```

:floppy_disk: [branch 07-collections-1](https://github.com/inkognitro/react-app-tutorial-code/compare/06-form-2...07-collections-1)



:floppy_disk: [branch 07-collections-2](https://github.com/inkognitro/react-app-tutorial-code/compare/07-collections-1...07-collections-2)

[« previous](06-form.md) | [next »](08-apiv1.md)