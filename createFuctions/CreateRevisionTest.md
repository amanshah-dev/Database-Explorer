CREATE FUNCTION "CreateRevisionTest"(userid INTEGER, numquestions INTEGER) RETURNS INTEGER
    LANGUAGE plpgsql
AS
$$
DECLARE
    testId INTEGER;
    questionIds INTEGER[];
    additionalQuestionIds INTEGER[];
    duplicateQuestionIds INTEGER[];
    totalSelected INTEGER := 0;
    remainingQuestions INTEGER;
    currentDate TEXT;
    testName TEXT;
    totalCorrect INTEGER := ROUND(numQuestions * 0.7); -- 70% of questions will be correct
    totalIncorrect INTEGER := numQuestions - totalCorrect; -- Remaining 30% will be incorrect
BEGIN
    -- Formatting the test name with today's date
    currentDate := TO_CHAR(now(), 'DD-Mon-YY');
    testName := 'Revision DPP - ' || currentDate;

    -- Creating a new Test entry
    INSERT INTO "Test"("name", "userId", "numQuestions", "createdAt", "updatedAt")
    VALUES (testName, userId, numQuestions, now(), now())
    RETURNING "id" INTO testId;

    -- Selecting questions for correct answers with specified weightages
    questionIds := '{}';
    questionIds := array_cat(questionIds, "SelectQuestionForRevisionTest"(userId, 75, 100, TRUE, ROUND(totalCorrect * 0.5)::INTEGER));
    questionIds := array_cat(questionIds, "SelectQuestionForRevisionTest"(userId, 50, 75, TRUE, ROUND(totalCorrect * 0.25)::INTEGER));
    questionIds := array_cat(questionIds, "SelectQuestionForRevisionTest"(userId, 25, 50, TRUE, ROUND(totalCorrect * 0.15)::INTEGER));
    remainingQuestions := totalCorrect - array_length(questionIds, 1);
    IF remainingQuestions > 0 THEN
        questionIds := array_cat(questionIds, "SelectQuestionForRevisionTest"(userId, 0, 25, TRUE, remainingQuestions));
    END IF;

    -- Selecting questions for incorrect answers
    questionIds := array_cat(questionIds, "SelectQuestionForRevisionTest"(userId, 75, 100, FALSE, ROUND(totalIncorrect * 0.5)::INTEGER));
    questionIds := array_cat(questionIds, "SelectQuestionForRevisionTest"(userId, 50, 75, FALSE, ROUND(totalIncorrect * 0.25)::INTEGER));
    questionIds := array_cat(questionIds, "SelectQuestionForRevisionTest"(userId, 25, 50, FALSE, ROUND(totalIncorrect * 0.15)::INTEGER));
    remainingQuestions := totalIncorrect - (array_length(questionIds, 1) - totalCorrect);
    IF remainingQuestions > 0 THEN
        questionIds := array_cat(questionIds, "SelectQuestionForRevisionTest"(userId, 0, 100, FALSE, remainingQuestions));
    END IF;

    -- Handling duplicate questions
    SELECT array_agg("questionId2") INTO duplicateQuestionIds FROM "DuplicateQuestion" WHERE "questionId1" = ANY(questionIds);
    questionIds := array(SELECT DISTINCT(unnest(questionIds)) EXCEPT SELECT DISTINCT(unnest(duplicateQuestionIds)));

    totalSelected := array_length(questionIds, 1);
    remainingQuestions := numQuestions - totalSelected;

    -- Fetch additional questions if not enough selected
    WHILE remainingQuestions > 0 LOOP
        -- Adjust the parameters as necessary to exclude already selected questionIds
        additionalQuestionIds := "SelectQuestionForRevisionTest"(userId, 0, 100, TRUE, remainingQuestions);

        -- Append newly fetched questionIds to the main array
        questionIds := array_cat(questionIds, additionalQuestionIds);

        -- Check for and remove duplicates in the combined list
        SELECT array_agg("questionId2") INTO duplicateQuestionIds FROM "DuplicateQuestion" WHERE "questionId1" = ANY(questionIds);
        questionIds := array(SELECT DISTINCT(unnest(questionIds)) EXCEPT SELECT DISTINCT(unnest(duplicateQuestionIds)));

        -- Update counters
        totalSelected := array_length(questionIds, 1);
        remainingQuestions := numQuestions - totalSelected;
    END LOOP;

    -- Randomizing questionIds
    questionIds := array(SELECT qId FROM unnest(questionIds) qId ORDER BY RANDOM());

    -- Inserting all question IDs in one go
    INSERT INTO "TestQuestion"("testId", "questionId", "createdAt", "updatedAt")
    SELECT testId, unnest(questionIds), now(), now();

    RETURN testId;
END;
$$;

ALTER FUNCTION "CreateRevisionTest"(INTEGER, INTEGER) OWNER TO learner;
