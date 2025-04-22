create function update_user_score_booster_snapshot_v2(p_userid integer, p_scheduleid integer DEFAULT NULL::integer) returns void
    language plpgsql
as
$$
DECLARE
    sched_id integer; -- Changed to integer to match function parameter type
    result_count INTEGER;
BEGIN
    -- If a specific scheduleId is provided
    IF p_scheduleid IS NOT NULL THEN
        -- Insert directly without using a temporary table
        INSERT INTO "UserScoreBoosterSnapshot" (
            "userId",
            "scheduleId",
            "scheduleName",
            "scheduleItemId",
            "scheduleItemName",
            "scheduledAt",
            "topicId",
            "topicName",
            "subjectId",
            "subjectName",
            questions,
            "correctQuestions",
            "assetId",
            "assetName",
            "assetLink",
            "platformQuestions",
            "platformCorrectQuestions",
            "createdAt",
            "updatedAt",
            "lastOverallUpdate"
        )
        SELECT DISTINCT
            "userId",
            "scheduleId",
            "scheduleName",
            "scheduleItemId",
            "scheduleItemName",
            "scheduledAt",
            "topicId",
            "topicName",
            "subjectId",
            "subjectName",
            questions,
            "correctQuestions",
            "assetId",
            "assetName",
            "assetLink",
            "platformQuestions",
            "platformCorrectQuestions",
            CURRENT_TIMESTAMP,  -- createdAt
            CURRENT_TIMESTAMP,  -- updatedAt
            CURRENT_TIMESTAMP   -- lastOverallUpdate
        FROM "GetUserScoreBoosterDetail"(p_userid::bigint, p_scheduleid::bigint)
        ON CONFLICT ("userId", "scheduleId", "scheduleItemId", "topicId", "assetId")
        DO UPDATE SET
            "scheduleName" = EXCLUDED."scheduleName",
            "scheduleItemName" = EXCLUDED."scheduleItemName",
            "scheduledAt" = EXCLUDED."scheduledAt",
            "topicName" = EXCLUDED."topicName",
            "subjectId" = EXCLUDED."subjectId",
            "subjectName" = EXCLUDED."subjectName",
            questions = EXCLUDED.questions,
            "correctQuestions" = EXCLUDED."correctQuestions",
            "assetName" = EXCLUDED."assetName",
            "assetLink" = EXCLUDED."assetLink",
            "platformQuestions" = EXCLUDED."platformQuestions",
            "platformCorrectQuestions" = EXCLUDED."platformCorrectQuestions",
            "updatedAt" = CURRENT_TIMESTAMP,  -- Update timestamp
            "lastOverallUpdate" = CURRENT_TIMESTAMP;
        
        -- Count and log results (moved after the insert)
        GET DIAGNOSTICS result_count = ROW_COUNT;
        RAISE NOTICE 'Processing schedule % with % records', p_scheduleid, result_count;
    ELSE
        -- If no scheduleId provided, process all score booster schedules for the user
        FOR sched_id IN
            SELECT 
                us."externalScheduleId"::integer -- Explicitly cast to integer
            FROM "UserSchedule" us
            JOIN "Schedule" s ON us."externalScheduleId" = s.id
            WHERE us."userId" = p_userid
            AND s."isScoreBooster" = true
        LOOP
            -- Process each schedule individually using recursion with explicit parameter types
            PERFORM public.update_user_score_booster_snapshot_v2(p_userid::integer, sched_id::integer);
        END LOOP;
    END IF;
END;
$$;

alter function update_user_score_booster_snapshot_v2(integer, integer) owner to learner;
