create function report_get_daily_users_for_mistakes(start_date date DEFAULT (CURRENT_DATE - '15 days'::interval), end_date date DEFAULT CURRENT_DATE)
    returns TABLE(created_date date, user_count bigint, total_count bigint)
    language plpgsql
as
$$
BEGIN
    RETURN QUERY
    SELECT
        "createdAt"::DATE AS created_date,
        COUNT(distinct("userId")) AS user_count,
        COUNT(*) as total_count
    FROM
        "UserMistake"
    WHERE
        "createdAt"::DATE BETWEEN start_date AND end_date  -- Filters tests within the date range
    GROUP BY
        "createdAt"::DATE
    ORDER BY
        "createdAt"::DATE DESC
    LIMIT 15;  -- Limits the number of rows to 15
END;
$$;

alter function report_get_daily_users_for_mistakes(date, date) owner to neetprep_rw;
