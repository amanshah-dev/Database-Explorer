CREATE FUNCTION get_score_booster_test_ids(p_user_id INTEGER)
    RETURNS TABLE(test_id INTEGER)
    LANGUAGE plpgsql
AS
$$
DECLARE
    v_cache_timestamp TIMESTAMP;
BEGIN
    -- Create cache table if not exists
    CREATE TEMPORARY TABLE IF NOT EXISTS score_booster_test_ids_cache (
        user_id INTEGER PRIMARY KEY,
        test_ids INTEGER[],
        created_at TIMESTAMP DEFAULT NOW()
    ) ON COMMIT PRESERVE ROWS;

    -- Check if cache exists and is less than 2 hours old
    SELECT created_at INTO v_cache_timestamp
    FROM score_booster_test_ids_cache
    WHERE user_id = p_user_id;

    -- If no cache or cache is older than 2 hours, refresh cache
    IF v_cache_timestamp IS NULL OR v_cache_timestamp < (NOW() - INTERVAL '2 hours') THEN
        -- Delete existing entry for this user
        DELETE FROM score_booster_test_ids_cache WHERE user_id = p_user_id;

        -- Insert new cache entry
        INSERT INTO score_booster_test_ids_cache (user_id, test_ids)
        SELECT
            p_user_id,
            ARRAY(
                SELECT DISTINCT REGEXP_REPLACE("assetLink", '^.*?/(\d+)$', '\1')::INTEGER
                FROM "ScheduleItemAsset" sia
                WHERE sia."assetLink" LIKE '%/neet-test/%'
                  AND sia."scheduleItemChapterId" IN (
                    SELECT id FROM "ScheduleItemChapter" sic
                    WHERE sic."scheduleItemId" = ANY(
                        SELECT id FROM "ScheduleItem"
                        WHERE "scheduleId" = ANY(
                            SELECT id FROM "Schedule"
                            WHERE "isScoreBooster" = true
                              AND id IN (
                                SELECT "externalScheduleId"
                                FROM "UserSchedule"
                                WHERE "userId" = p_user_id
                              )
                            )
                        )
                    )
            );
    END IF;

    -- Return the cached test IDs
    RETURN QUERY
        SELECT unnest(test_ids)
        FROM score_booster_test_ids_cache
        WHERE user_id = p_user_id;
END;
$$;

ALTER FUNCTION get_score_booster_test_ids(integer) OWNER TO learner;
