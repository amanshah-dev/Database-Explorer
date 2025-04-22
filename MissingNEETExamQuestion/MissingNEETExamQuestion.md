CREATE FUNCTION "MissingNEETExamQuestion"(yr INTEGER)
RETURNS TABLE("qId" INTEGER)
LANGUAGE plpgsql
AS
$$
  BEGIN
  RETURN QUERY 
  SELECT "questionId" 
  FROM (
      SELECT * 
      FROM (
        SELECT "CourseTestQuestion"."questionId" 
        FROM "CourseTestQuestion", "Test" 
        WHERE "courseId" = 980 
          AND "Test"."id" = "CourseTestQuestion"."testId" 
          AND EXTRACT(YEAR FROM "startedAt") = yr
        EXCEPT
        SELECT "CourseQuestion"."questionId" 
        FROM "CourseQuestion", "Question" 
        WHERE "courseId" = 8 
          AND "CourseQuestion"."questionId" = "Question"."id" 
          AND EXISTS (
            SELECT * 
            FROM "QuestionDetail" 
            WHERE "QuestionDetail"."questionId" = "Question"."id" 
              AND "exam" IN ('NEET', 'AIPMT') 
              AND "year" = yr
          ) 
      ) a
      EXCEPT 
      SELECT "DuplicateQuestion"."questionId2" 
      FROM "CourseQuestion", "Question", "DuplicateQuestion" 
      WHERE "CourseQuestion"."courseId" = 8 
        AND "questionId" = "Question"."id" 
        AND EXISTS (
          SELECT * 
          FROM "QuestionDetail" 
          WHERE "QuestionDetail"."questionId" = "Question"."id" 
            AND "exam" IN ('NEET', 'AIPMT') 
            AND "year" = yr
        ) 
        AND "DuplicateQuestion"."questionId1" = "Question"."id"
      ) b
      EXCEPT 
      SELECT "DuplicateQuestion"."questionId1" AS "questionId" 
      FROM "CourseQuestion", "Question", "DuplicateQuestion" 
      WHERE "CourseQuestion"."courseId" = 8 
        AND "questionId" = "Question"."id" 
        AND EXISTS (
          SELECT * 
          FROM "QuestionDetail" 
          WHERE "QuestionDetail"."questionId" = "Question"."id" 
            AND "exam" IN ('NEET', 'AIPMT') 
            AND "year" >= yr
        ) 
        AND "DuplicateQuestion"."questionId2" = "Question"."id";
  END
$$;

ALTER FUNCTION "MissingNEETExamQuestion"(INTEGER) OWNER TO learner;
