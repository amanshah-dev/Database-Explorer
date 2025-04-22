-- Create function to clone tests from the main test
create function clonetestsfrommegatest(maintestid integer, testpatternid integer) returns void
    language plpgsql
as
$$
DECLARE
    sectionRecord RECORD;            -- Variable to hold each section's record
    newTestId INTEGER;               -- Variable to store the new test ID
    newTestQuestions INTEGER[];      -- Variable to store questions for the new test
BEGIN
    -- Loop over each ChapterSection entry
    FOR sectionRecord IN SELECT * FROM "ChapterSection" LOOP
        -- Create a new test for the current chapter and section
        INSERT INTO "Test" ("name", "description", "instructions", "numQuestions", "durationInMin", "free", "sections", "positiveMarks", "negativeMarks", "userId")
        VALUES (
            'NEW NCERT SYLLABUS - TARGET QUESTION BANK - Chapter ID - ' || sectionRecord."chapterId" || ' Section - ' || sectionRecord."section",
            'NEW NCERT SYLLABUS - TARGET QUESTION BANK - Chapter ID - ' || sectionRecord."chapterId" || ' Section - ' || sectionRecord."section" || ' ID: ' || sectionRecord."id",
            'Follow the instructions carefully.',
            NULL,  -- Number of questions is set to NULL
            NULL,  -- Duration in minutes is set to NULL, could be customized
            FALSE, -- All tests are free
            NULL,  -- Sections are not provided as JSON, could be added if needed
            4,     -- Positive marks per question
            1,     -- Negative marks per question
            97     -- Hardcoded user ID for now
        ) RETURNING "id" INTO newTestId;  -- Store the new test ID

        -- Insert easy questions into new test if Section is A
        IF sectionRecord."section"::TEXT LIKE '%-Section-A' THEN
            -- Select easy questions (more than 50% correct percentage) for Section A
            SELECT array_agg("questionId")
            INTO newTestQuestions
            FROM "TestQuestion"
            JOIN "Question" ON "testId" = maintestid AND "Question"."id" = "TestQuestion"."questionId" AND "topicId" = sectionRecord."chapterId"
            LEFT JOIN "QuestionAnalytics" ON "QuestionAnalytics"."id" = "TestQuestion"."questionId" AND "correctPercentage" > 50;
            -- Insert easy questions into the new test
            INSERT INTO "TestQuestion" ("testId", "questionId")
            SELECT newTestId, unnest(newTestQuestions);
        END IF;

        -- Insert hard questions into new test if Section is B
        IF sectionRecord."section"::TEXT LIKE '%-Section-B' THEN
            -- Select hard questions (less than 50% correct percentage) for Section B
            SELECT array_agg("questionId")
            INTO newTestQuestions
            FROM "TestQuestion"
            JOIN "Question" ON "testId" = maintestid AND "Question"."id" = "TestQuestion"."questionId" AND "topicId" = sectionRecord."chapterId"
            LEFT JOIN "QuestionAnalytics" ON "QuestionAnalytics"."id" = "TestQuestion"."questionId" AND ("correctPercentage" < 50 or "correctPercentage" is null);
            -- Insert hard questions into the new test
            INSERT INTO "TestQuestion" ("testId", "questionId")
            SELECT newTestId, unnest(newTestQuestions);
        END IF;

        -- Insert the new test into the QuesBankTest table
        INSERT INTO "QuesBankTest" ("testId", "chapterId", "section", "seqId", "testPatternId")
        VALUES (newTestId, sectionRecord."chapterId", sectionRecord."section", 10000, testpatternid);
    END LOOP;
END;
$$;

-- Alter function owner
alter function clonetestsfrommegatest(integer, integer) owner to neetprep_rw;
