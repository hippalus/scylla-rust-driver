# Paged query
Sometimes query results might be so big that one prefers not to fetch them all at once,
e.g. to reduce latency and/or memory footprint.
Paged queries allow to receive the whole result page by page, with a configurable page size.

`Session::query_iter` and `Session::execute_iter` take a [simple query](simple.md)
or a [prepared query](prepared.md) and return an `async` iterator over result `Rows`.

> ***Warning***\
> In case of unprepared variant (`Session::query_iter`) if the values are not empty
> driver will first fully prepare a query (which means issuing additional request to each
> node in a cluster). This will have a performance penalty - how big it is depends on
> the size of your cluster (more nodes - more requests) and the size of returned
> result (more returned pages - more amortized penalty). In any case, it is preferable to
> use `Session::execute_iter`.

### Examples
Use `query_iter` to perform a [simple query](simple.md) with paging:
```rust
# extern crate scylla;
# extern crate futures;
# use scylla::Session;
# use std::error::Error;
# async fn check_only_compiles(session: &Session) -> Result<(), Box<dyn Error>> {
use scylla::IntoTypedRows;
use futures::stream::StreamExt;

let mut rows_stream = session
    .query_iter("SELECT a, b FROM ks.t", &[])
    .await?
    .into_typed::<(i32, i32)>();

while let Some(next_row_res) = rows_stream.next().await {
    let (a, b): (i32, i32) = next_row_res?;
    println!("a, b: {}, {}", a, b);
}
# Ok(())
# }
```

Use `execute_iter` to perform a [prepared query](prepared.md) with paging:
```rust
# extern crate scylla;
# extern crate futures;
# use scylla::Session;
# use std::error::Error;
# async fn check_only_compiles(session: &Session) -> Result<(), Box<dyn Error>> {
use scylla::IntoTypedRows;
use scylla::prepared_statement::PreparedStatement;
use futures::stream::StreamExt;

let prepared: PreparedStatement = session
    .prepare("SELECT a, b FROM ks.t")
    .await?;

let mut rows_stream = session
    .execute_iter(prepared, &[])
    .await?
    .into_typed::<(i32, i32)>();

while let Some(next_row_res) = rows_stream.next().await {
    let (a, b): (i32, i32) = next_row_res?;
    println!("a, b: {}, {}", a, b);
}
# Ok(())
# }
```

Query values can be passed to `query_iter` and `execute_iter` just like in a [simple query](simple.md)

### Configuring page size
It's possible to configure the size of a single page.

On a `Query`:
```rust
# extern crate scylla;
# use scylla::Session;
# use std::error::Error;
# async fn check_only_compiles(session: &Session) -> Result<(), Box<dyn Error>> {
use scylla::query::Query;

let mut query: Query = Query::new("SELECT a, b FROM ks.t");
query.set_page_size(16.try_into().unwrap());

let _ = session.query_iter(query, &[]).await?; // ...
# Ok(())
# }
```

On a `PreparedStatement`:
```rust
# extern crate scylla;
# use scylla::Session;
# use std::error::Error;
# async fn check_only_compiles(session: &Session) -> Result<(), Box<dyn Error>> {
use scylla::prepared_statement::PreparedStatement;

let mut prepared: PreparedStatement = session
    .prepare("SELECT a, b FROM ks.t")
    .await?;

prepared.set_page_size(16.try_into().unwrap());

let _ = session.execute_iter(prepared, &[]).await?; // ...
# Ok(())
# }
```

### Passing the paging state manually
It's possible to fetch a single page from the table, extract the paging state
from the result and manually pass it to the next query. That way, the next
query will start fetching the results from where the previous one left off.

On a `Query`:
```rust
# extern crate scylla;
# use scylla::Session;
# use std::error::Error;
# async fn check_only_compiles(session: &Session) -> Result<(), Box<dyn Error>> {
use scylla::query::Query;
use scylla::statement::{PagingState, PagingStateResponse};
use std::ops::ControlFlow;

let paged_query = Query::new("SELECT a, b, c FROM ks.t").with_page_size(6.try_into().unwrap());

let mut paging_state = PagingState::start();
loop {
    let (res, paging_state_response) = session
        .query_single_page(paged_query.clone(), &[], paging_state)
        .await?;

    // Do something with `res`.
    // ...

    match paging_state_response.into_paging_control_flow() {
        ControlFlow::Break(()) => {
            // No more pages to be fetched.
            break;
        }
        ControlFlow::Continue(new_paging_state) => {
            // Update paging state from the response, so that query
            // will be resumed from where it ended the last time.
            paging_state = new_paging_state
        }
    }
}

# Ok(())
# }
```

> ***Warning***\
> If the values are not empty, driver first needs to send a `PREPARE` request
> in order to fetch information required to serialize values. This will affect
> performance because 2 round trips will be required instead of 1.

On a `PreparedStatement`:
```rust
# extern crate scylla;
# use scylla::Session;
# use std::error::Error;
# async fn check_only_compiles(session: &Session) -> Result<(), Box<dyn Error>> {
use scylla::query::Query;
use scylla::statement::{PagingState, PagingStateResponse};
use std::ops::ControlFlow;

let paged_prepared = session
    .prepare(Query::new("SELECT a, b, c FROM ks.t").with_page_size(7.try_into().unwrap()))
    .await?;

let mut paging_state = PagingState::start();
loop {
    let (res, paging_state_response) = session
        .execute_single_page(&paged_prepared, &[], paging_state)
        .await?;

    println!(
        "Paging state response from the prepared statement execution: {:#?} ({} rows)",
        paging_state_response,
        res.rows_num()?,
    );

    match paging_state_response.into_paging_control_flow() {
        ControlFlow::Break(()) => {
            // No more pages to be fetched.
            break;
        }
        ControlFlow::Continue(new_paging_state) => {
            // Update paging state from the response, so that query
            // will be resumed from where it ended the last time.
            paging_state = new_paging_state
        }
    }
}
# Ok(())
# }
```

### Performance
Performance is the same as in non-paged variants.\
For the best performance use [prepared queries](prepared.md).