create procedure update_subject_expert(IN subjectname text, IN achievementcategoryid bigint, IN userids bigint[] DEFAULT ARRAY[]::bigint[])
    language plpgsql
as
$$
DECLARE
    achievement_record RECORD;
    eligibleUsers      BIGINT[] := ARRAY []::BIGINT[];
BEGIN
    FOR achievement_record IN
        SELECT "id", "currentLevelUnit" AS current_level, "achievementCategoryId"
        FROM "Achievement"
        WHERE "achievementCategoryId" = achievementCategoryId
        ORDER BY current_level
        LOOP
            -- Fetch eligible users dynamically
            eligibleUsers := bestSubjectScorers(userIds, subjectName, achievement_record.current_level);
            CALL insert_user_achievements(eligibleUsers, achievement_record);
        END LOOP;
END;
$$;

alter procedure update_subject_expert(text, bigint, bigint[]) owner to learner;
