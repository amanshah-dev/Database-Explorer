CREATE FUNCTION "DeleteQuestionFromPGCourse"(questionids TEXT) 
    RETURNS VOID
    LANGUAGE plpgsql
AS
$$
DECLARE
  testId INT;
BEGIN
  -- Loop through all testIds associated with courseId 2135
  FOR testId IN
    SELECT "testId" 
    FROM "CourseTest" 
    WHERE "courseId" = 2135
  LOOP
    -- Call the "DeleteTestQuestionsAndUpdateTest" function for each testId
    PERFORM "DeleteTestQuestionsAndUpdateTest"(testId, questionIds);
  END LOOP;
END;
$$;

ALTER FUNCTION "DeleteQuestionFromPGCourse"(TEXT) OWNER TO learner;
