## Problems in `get_awarded_transactions.sql`

| Issue | Effect |
|---|---|
| It doesn't select `participantfullname` | `Row.jsx` uses that field for the tooltip, and the default sort (`contractsize,participantfullname`) uses it as the tie-breaker. The field is always undefined, so the tooltip is empty and the tie-breaker does nothing. |
| No `ORDER BY` | Row order is undefined, so any kind of paging would be unstable. |
| No limit or offset parameters | The database can't do the paging. Suggest using a Cursor-based pagination istead of traditional offset and limit pagination: full table scan |
| It returns 6 columns the UI never shows (`iso`, `ftrparticipant`, `auctiontype`, `counterflow`, `hours`, `hrs_in_period`) | Wasted payload. |
| It uses a refcursor, a single-quoted body, and the default `VOLATILE` | A plain set-returning function marked `STABLE` would be simpler. |


Proposed solution using pagination with limit and offset. For 5_000 rows this is an acceptable solution, but for larger datasets, consider using cursor based pagination instead of traditional offset and limit pagination to avoid full table scans.

```sql 

DROP FUNCTION IF EXISTS public.get_awarded_transactions();
DROP FUNCTION IF EXISTS public.get_awarded_transactions(integer, integer);

CREATE FUNCTION public.get_awarded_transactions(p_limit integer, p_offset integer)
RETURNS TABLE (
    participantshortname varchar, participantfullname varchar,
    tradetype varchar, peaktype varchar, hedgetype varchar,
    sourceid bigint, sinkid bigint,
    contractsize real, cost real, revenue real, profit real,
    total_count bigint
)
LANGUAGE sql STABLE AS $$
    SELECT p.participantshortname, p.participantfullname,
           t.tradetype, t.peaktype, t.hedgetype,
           t.sourceid, t.sinkid,
           t.contractsize, t.cost, t.revenue, t.profit,
           count(*) OVER () AS total_count
    FROM trans t
    JOIN participants p ON p.ftrparticipant = t.ftrparticipant
    ORDER BY t.contractsize, p.participantfullname, t.sourceid, t.sinkid
    LIMIT p_limit OFFSET p_offset
$$;

```