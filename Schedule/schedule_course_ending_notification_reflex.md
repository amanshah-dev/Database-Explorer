create function schedule_course_ending_notification_reflex() 
    returns void 
    language plpgsql 
as 
$$
BEGIN 
    WITH filteredUsers AS ( 
        SELECT 
            uec."userId", 
            MAX(uec."expiryAt") AS "expiryAt" 
        FROM 
            "UserCourse" uec 
        WHERE 
            uec."courseId" = 2135 
            AND uec."expiryAt" BETWEEN CURRENT_DATE + INTERVAL '2 day' AND CURRENT_DATE + INTERVAL '3 days' 
        GROUP BY 
            uec."userId" 
    ), 
    eligibleUsers AS ( 
        SELECT 
            fu."userId", 
            fu."expiryAt" 
        FROM 
            filteredUsers fu 
        WHERE NOT EXISTS ( 
            SELECT 1 
            FROM "UserCourse" uc2 
            WHERE uc2."userId" = fu."userId" 
            AND uc2."courseId" = 2135 
            AND uc2."expiryAt" > fu."expiryAt" 
        ) 
    ), 
    alreadyScheduleNotifi AS ( 
        SELECT DISTINCT 
            "userId" 
        FROM 
            "Notification" 
        WHERE 
            "contextType" = 'Course Ending Nudge Reflex' 
            AND "scheduledAt" BETWEEN CURRENT_TIMESTAMP AND CURRENT_TIMESTAMP + INTERVAL '24 hours' 
            AND "id" > 312384763 
            AND "scheduledAt" IS NOT NULL 
    ) 

    -- Insert notifications for eligible users with expiring courses
    INSERT INTO "Notification" ("userId", "title", "body", "scheduledAt", "contextType") 
    SELECT 
        eu."userId", 
        'Attention: Reflex Subscription Expiry!', -- Updated title
        'Your subscription expires tomorrow. Renew Now to continue using Reflex.', -- Updated body
        (eu."expiryAt"::date - INTERVAL '1 day') + INTERVAL '4 hours' + (FLOOR(RANDOM() * 60) + 1) * INTERVAL '1 minute' AS "scheduledAt", -- Updated interval with random minutes
        'Course Ending Nudge Reflex' 
    FROM 
        eligibleUsers eu 
    WHERE 
        eu."userId" NOT IN (SELECT "userId" FROM alreadyScheduleNotifi) 
    ON CONFLICT ("scheduledAt", "userId", "contextId", "contextType") 
    WHERE "id" > 312384763 AND "scheduledAt" IS NOT NULL 
    DO NOTHING; 

END; 
$$;

alter function schedule_course_ending_notification_reflex() owner to neetprep_rw;
