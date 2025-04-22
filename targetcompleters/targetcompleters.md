create function targetcompleters(userids bigint[], target integer) returns bigint[]
    language plpgsql
as
$$
DECLARE
    targetCompleters BIGINT[] := ARRAY []::BIGINT[]; -- Array to store the result
BEGIN
    SELECT ARRAY_AGG("userId")
    INTO targetCompleters
    FROM (SELECT ta."userId"
          FROM "TestAttempt" ta
                   join "Target" t on ta."testId" = t."testId"
          WHERE ta."userId" = ANY (userIds)
            and (ta.result ->> 'totalMarks')::INT >= t.score
--               ta."createdAt" > CURRENT_DATE - interval '1 month' and
          GROUP BY ta."userId"
          HAVING COUNT(*) >= target) subquery;
    return targetCompleters;
END;
$$;

alter function targetcompleters(bigint[], integer) owner to neetprep_rw;
