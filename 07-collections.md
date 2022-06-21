[« previous](06-form.md) | [next »](08-apiv1.md)

## 7. Collections
In this chapter we are going to create a provider for entity arrays which
can be passed as prop (not only) to table or dropdown components.
I am sure we can work out a way to standardize the process for fetching entities
independent of where they are stored and provided:
Either from an API endpoint or just from a simple array.

### 7.1 Understand the principles
First, I'd like to introduce some terms of the Java world in relation to an array of objects:

> The Collection in Java is a framework that provides an architecture to store and manipulate the group of objects.
> 
> Java Collections can achieve all the operations that you perform on a data such as searching, sorting, insertion,
> manipulation, and deletion.
> 
> `https://www.javatpoint.com/collections-in-java, 2022-06-21`

As of this, I think it's a better idea to call the data itself - without any functionality - the "collection".
The processor of operations like fetching, creating etc. could be called "collection manager"
or in case of just providing collections, the "collection provider".
Java does not make a difference about these two.
This aligns also with the naming in the React-Admin framework.

> :bulb: Did you know?
> Marmelab's React-Admin uses a similar pattern called
> [data providers](https://marmelab.com/react-admin/DataProviders.html).
> Even the creation, update and deletion of one or multiple entities is supported.
>
> `https://www.javatpoint.com/collections-in-java, 2022-06-21`

Let's only support fetching with filters and sorting params by a "collection provider".
Pagination is also relevant when hundreds of entities are available.
As of my experience in frontend programming, writing and storing data is often an 
individually designed process. Because of this we'll not support write operations in our collection providers,
or at least for now.

### 7.2 Define the interface
Let's try to create an interface for the collection provider which takes a query holding
the information about filtering, sorting and pagination params.

```typescript jsx
//src/packages/core/collection/collection.ts

export type Sorting = {
    field: string;
    direction: 'asc' | 'desc';
}[];

export type Filters = {
    [field: string]: string | number | boolean;
};

// Let's wrap the data in an Entry holding a key:
// Not every entity could have a primary key called "id".
// Therefore we need to be able to create a custom and recognizable key.
export type Entry<Data = any> = {
    key: string;
    data: Data;
};

// We should support pagination by limit and offset.
// This keeps the door open for every data reload constellation.
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

// Furthermore we should provide a collection info object.
// With this we store for which filters, sorting and pagination params the delivered entries
// are really for.
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
// state of a collection provider. With this we could for instance show a loader icon.
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



:floppy_disk: [branch 07-collections-1](https://github.com/inkognitro/react-app-tutorial-code/compare/06-form-2...07-collections-1)

:floppy_disk: [branch 07-collections-2](https://github.com/inkognitro/react-app-tutorial-code/compare/07-collections-1...07-collections-2)

[« previous](06-form.md) | [next »](08-apiv1.md)