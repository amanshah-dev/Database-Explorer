create function report_get_daily_custom_tests(max_num_questions integer DEFAULT 200, start_date date DEFAULT (CURRENT_DATE - '15 days'::interval), end_date date DEFAULT CURRENT_DATE)
    returns TABLE(created_date date, test_count bigint)
    language plpgsql
as
$$
BEGIN
    RETURN QUERY
    SELECT
        "createdAt"::DATE AS created_date,
        COUNT(*) AS test_count
    FROM
        "Test"
    WHERE
        "name" ~ 'Custom Practice Test '  -- Filters tests by name pattern
        AND "numQuestions" <= max_num_questions  -- Filters tests by maximum number of questions
        AND "createdAt"::DATE BETWEEN start_date AND end_date  -- Filters tests within the date range
        AND "userId" IS NOT NULL
    GROUP BY
        "createdAt"::DATE
    ORDER BY
        "createdAt"::DATE DESC
    LIMIT 15;  -- Limits the number of rows to 15
END;
$$;

alter function report_get_daily_custom_tests(integer, date, date) owner to neetprep_rw;
