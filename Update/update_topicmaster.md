create procedure update_topicmaster(IN achievementcategoryid bigint, IN userids bigint[] DEFAULT ARRAY[]::bigint[])
    language plpgsql
as
$$
DECLARE
    achievement_record RECORD;
    eligibleUsers      BIGINT[];
BEGIN

    FOR achievement_record IN
        /* Query to get user answers and match them with correct options in the Question table */
        SELECT "id", "currentLevelUnit" as current_level, "achievementCategoryId"
        FROM "Achievement"
        WHERE "achievementCategoryId" = achievementCategoryId
        order by current_level
        LOOP
            eligibleUsers := calculate_topic_master_proficients(userIds, achievement_record.current_level);
            call insert_user_achievements(eligibleUsers, achievement_record);
        end loop;
END;
$$;

alter procedure update_topicmaster(bigint, bigint[]) owner to neetprep_rw;
