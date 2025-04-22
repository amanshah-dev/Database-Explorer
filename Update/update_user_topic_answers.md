create function update_user_topic_answers(userids bigint[] DEFAULT ARRAY[]::bigint[]) returns void
    language plpgsql
as
$$
DECLARE
    lastSynced   DATE;
    currentArray BOOLEAN[];
    answerRecord RECORD;
    n            INTEGER;
BEGIN
    /* Fetch the most recent sync timestamp from UserTopicAnswers */
    SELECT COALESCE(MAX(uta."lastUpdated"), '2024-05-05')
    INTO lastSynced
    FROM "UserTopicAnswers" uta;

    /* Loop through each result from the Answer table after the last sync timestamp */
    FOR answerRecord IN
        /* Query to get user answers and match them with correct options in the Question table */
        SELECT a."userId"                                                                  as userId,
               q."topicId"                                                                 as topicId,
               ARRAY_AGG((a."userAnswer" = q."correctOptionIndex") ORDER BY a."createdAt") AS correctArray
        FROM "Answer" a
                 JOIN
             "Question" q ON a."questionId" = q."id"
        WHERE a."createdAt" > lastSynced
          AND q."topicId" IS NOT NULL
          AND a."userId" = ANY (userIds)
        GROUP BY a."userId", q."topicId"
        LOOP
            -- Logging for debugging purposes
            RAISE NOTICE 'Processing UserId: %, TopicId: %, v_correctArray: %',
                answerRecord.userId, answerRecord.topicId, answerRecord.correctArray;

            n := array_length(answerRecord.correctArray, 1);
            IF n > 100 THEN
                answerRecord.correctArray := answerRecord.correctArray[n - 99:]; -- Remove the oldest entry
            END IF;

            /* Retrieve the current "isCorrectArray" for the user and topic, locking the row for update */
            SELECT COALESCE("isCorrectArray", ARRAY []::BOOLEAN[])
            INTO currentArray
            FROM "UserTopicAnswers"
            WHERE "userId" = answerRecord.userId
              AND "topicId" = answerRecord.topicId
                FOR UPDATE;

            /* If no record exists for the user-topic combination, insert a new one */
            IF currentArray IS NULL OR array_length(currentArray, 1) = 0 THEN
                INSERT INTO "UserTopicAnswers" ("userId", "topicId", "isCorrectArray", "lastUpdated")
                VALUES (answerRecord.userId, answerRecord.topicId, answerRecord.correctArray, CURRENT_TIMESTAMP);
            ELSE
                /* Append the new result to the current array */
                currentArray := currentArray || answerRecord.correctArray;

                n := array_length(currentArray, 1);
                /* Ensure the array has a maximum of 100 elements by truncating the oldest */
                IF n > 100 THEN
                    currentArray := currentArray[n - 99:]; -- Remove the oldest entry
                END IF;

                /* Update the existing entry with the new array and timestamp */
                UPDATE "UserTopicAnswers"
                SET "isCorrectArray" = currentArray,
                    "lastUpdated"    = CURRENT_TIMESTAMP
                WHERE "userId" = answerRecord.userId
                  AND "topicId" = answerRecord.topicId;
            END IF;
        END LOOP;
END;
$$;

alter function update_user_topic_answers(bigint[]) owner to neetprep_rw;
