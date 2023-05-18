The unexpected test failures in my recent [pull requests](https://github.com/supabase/pg_net/actions/runs/4987953205/attempts/5), which only modified documentation, prompted an investigation. The root cause appears to be timeouts during test endpoint calls to https://httpbin.org. If the response exceeds the default 2-second timeout, `net._http_collect_response` returns a misleading success status:

```markdown
| status  | message | response |
| ------- | ------- | -------- |
| SUCCESS | ok      | (,,)     |
```
The expected return should include `<status_code>, <headers>, <body>` in the response field. A simple fix would be to adjust the [worker.c](https://github.com/supabase/pg_net/blob/master/src/worker.c) to yield an 'ERROR' code upon timeouts, but ongoing discussions around a potential breaking update in [issue 74](https://github.com/supabase/pg_net/issues/74) and [issue 62](https://github.com/supabase/pg_net/issues/62) suggest caution.

As a provisional solution, I ran two GET requests using the [Postman Echo API](https://learning.postman.com/docs/developer/echo-api/):

```sql
-- REQUEST_ID 81: TIMED OUT
SELECT net.http_get (
            'https://postman-echo.com/get?foo1=bar1&foo2=bar2',
            timeout_milliseconds := 1
) AS request_id;

-- REQUEST_ID 82: SUCCESSFUL
SELECT net.http_get (
            'https://postman-echo.com/get?foo1=bar1&foo2=bar2'
) AS request_id;
```

I discovered that [net._http_collect_response](https://github.com/supabase/pg_net/blob/f7ea986b8241a9adbe278d4868875650a3f7db36/sql/pg_net.sql#L280), while determining success status, overlooks error messages and assumes any recorded response denotes a successful execution. This erroneous assumption can be rectified with the following adjustments:



```sql

-- Collect responses of an http request
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
    
    --  If no response or an error exists, return ERROR
    -- BELOW LINE MODIFIED: ADDED OR CONDITION
    if rec is null OR rec.error_msg IS NOT NULL then 
        -- The request is either still processing or the request_id provided does not exist

        -- TODO: request in progress is indistinguishable from request that doesn't exist

        -- No request matching request_id found
        return (
            'ERROR',
            -- BELOW LINE MODIFIED: ADDED COALESCE
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


This proposed solution inspects the error message field for each request, and if a message is found or the request is absent, the function will return an 'ERROR' status.
