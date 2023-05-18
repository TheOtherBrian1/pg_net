I noticed that the tests were failing for my [last pull request ](https://github.com/supabase/pg_net/actions/runs/4987953205/attempts/5), as well as this one. Considering that I've only submitted documentation updates, it seemed strange to me that any code was failing. 

After looking into the issue, I believe the error was caused by timeouts. The tests generally call an endpoint from https://httpbin.org. However, if for any reason the results take more than 2 seconds to execute (happens often), the function will return back the following results:

> | status  | message | response |
> | ------- | ------- | -------- |
> | SUCCESS | ok      | (,,)     |

The results should be something like this:

> | status  | message | response |
> | ------- | ------- | -------- |
> | SUCCESS | ok      | (<status_code>, <headers>, <body>)     |

This issue can be solved by modifying the [worker.c](https://github.com/supabase/pg_net/blob/master/src/worker.c) to return an 'ERROR' code when the timeout fails. However, after coming the discussion in [issue 74](https://github.com/supabase/pg_net/issues/74) and the enhancement suggestion in [issue 62](https://github.com/supabase/pg_net/issues/62), I doubt it would be wise to make these changes while considerations for a breaking update is going on.

However, there may be a relatively simple solution. I made two get requests using the [Postman Echo API](https://learning.postman.com/docs/developer/echo-api/):

```sql
-- REQUEST_ID 81: TIMED OUT
SELECT net.http_get (
            'https://postman-echo.com/get?foo1=bar1&foo2=bar2',
            timeout_milliseconds := 1
) AS request_id;

-- REQUEST_ID 82: 
SELECT net.http_get (
            'https://postman-echo.com/get?foo1=bar1&foo2=bar2'
) AS request_id;
```
I then checked the requests' response entries in the http._http_response table:

| id | status_code | content_type                    | headers         | content                                                                                                                                                                                                                                                                                                                                                                                                  | timed_out | error_msg           | created                       |
|----|-------------|---------------------------------|-----------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------|---------------------|-------------------------------|
| 81 |             |                                 |                 |                                                                                                                                                                                                                                                                                                                                                                                                          |           | Timeout was reached | 2023-05-18 04:01:07.054579+00 |
| 82 | 200         | application/json; charset=utf-8 | [object Object] | `{ "args": { "foo1": "bar1", "foo2": "bar2" }, "headers": { "x-forwarded-proto": "https", "x-forwarded-port": "443", "host": "postman-echo.com", "x-amzn-trace-id": "Root=1-6465a4be-32a2128c278ce40d3879ff2f", "accept": "*/*", "api-key-header": "<API KEY>", "user-agent": "pg_net/0.7.1" }, "url": "https://postman-echo.com/get?foo1=bar1&foo2=bar2" }` | false     |                     | 2023-05-18 04:08:30.839851+00 |

Although the timeout response is clearly problematic, it appears that the [net._http_collect_response](https://github.com/supabase/pg_net/blob/f7ea986b8241a9adbe278d4868875650a3f7db36/sql/pg_net.sql#L280) , which interprets a response's success status, does not check the error message column to determine whether a request succeeded. It assumes that if the response was recorded at all, that the background worker successfully executed the request:

```sql

-- Collect respones of an http request
-- API: Private
create or replace function net._http_collect_response(
    -- request_id reference
    request_id bigint,
    -- when `true`, return immediately. when `false` wait for the request to complete before returning
    async bool default true
)
    -- http response composite wrapped in a result type
    returns net.http_response_result
    strict
    volatile
    parallel safe
    language plpgsql
    security definer
as $$
declare
    rec net._http_response;
    req_exists boolean;
begin

    if not async then
        perform net._await_response(request_id);
    end if;

    select *
    into rec
    from net._http_response
    where id = request_id;

    if rec is null then
        -- The request is either still processing or the request_id provided does not exist

        -- TODO: request in progress is indistinguishable from request that doesn't exist

        -- No request matching request_id found
        return (
            'ERROR',
            'request matching request_id not found',
            null
        )::net.http_response_result;

    end if;

    -- Return a valid, populated http_response_result
    return (
        'SUCCESS',
        'ok',
        (
            rec.status_code,
            rec.headers,
            rec.content
        )::net.http_response
    )::net.http_response_result;
end;
$$;
```

There are many ways this can be resolved, however, I'm going to suggest the following edits:

```sql

-- Collect responses of an http request
-- API: Private | IS THERE A REASON THIS IS STILL COMMENTED AS PRIVATE?
create or replace function net._http_collect_response(
    -- request_id reference
    request_id bigint,
    -- when `true`, return immediately. when `false` wait for the request to complete before returning
    async bool default true
)
    -- http response composite wrapped in a result type
    returns net.http_response_result
    strict
    volatile
    parallel safe
    language plpgsql
    security definer
as $$
declare
    rec net._http_response;
    req_exists boolean;
begin

    if not async then
        perform net._await_response(request_id);
    end if;

    select *
    into rec
    from net._http_response
    where id = request_id;
    
    -- MODIFIED LINE: ADDED OR 
    if rec is null OR rec.error_msg IS NOT NULL then
        -- The request is either still processing or the request_id provided does not exist

        -- TODO: request in progress is indistinguishable from request that doesn't exist

        -- No request matching request_id found
        return (
            'ERROR',
            -- MODIFIED LINE: ADDED COALESCE
            COALESCE(rec.error_msg, 'request matching request_id not found'),
            null
        )::net.http_response_result;

    end if;


    -- Return a valid, populated http_response_result
    return (
        'SUCCESS',
        'ok',
        (
            rec.status_code,
            rec.headers,
            rec.content
        )::net.http_response
    )::net.http_response_result;
end;
$$;
```
