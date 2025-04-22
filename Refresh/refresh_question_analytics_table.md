create function refresh_question_analytics_table(datelower timestamp with time zone, dateupper timestamp with time zone, questionid integer DEFAULT NULL::integer) returns void
    language plpgsql
as
$$
      DECLARE
      BEGIN
        INSERT INTO "QuestionAnalyticsTable"
          SELECT 
            "t".id,
            "t"."tagExist",
            "t"."correctAnswerCount",
            "t"."incorrectAnswerCount",
            "t"."option1AnswerCount",
            "t"."option2AnswerCount",
            "t"."option3AnswerCount",
            "t"."option4AnswerCount",
            "t"."incorrectReason1Count",
            "t"."incorrectReason2Count",
            "t"."incorrectReason3Count",
            get_correct_percentage( "t"."correctAnswerCount", "t"."incorrectAnswerCount") AS "correctPercentage",
            get_difficulty_level( "t"."correctAnswerCount", "t"."incorrectAnswerCount") AS "difficultyLevel",
            CASE 
              WHEN (
                EXISTS(
                  SELECT "Subject".id 
                  FROM public."TopicQuestion", public."Topic", public."Subject"
                  WHERE (
                    ("TopicQuestion"."topicId" = "t".id) AND 
                    ("TopicQuestion"."topicId" = "Topic".id) AND 
                    ("Topic"."subjectId" = "Subject".id) AND ("Subject"."courseId" = 8)
                  )
                )) THEN true
              ELSE false
            END AS "inFullCourse"
        FROM (SELECT 
          "Question".id,
          CASE
              WHEN (("Question".explanation ~~ '%<audio%'::text) OR ("Question".explanation ~~ '%<video%'::text)) THEN true
              ELSE false
          END AS "tagExist",
          count(
              CASE
                  WHEN ("Question"."correctOptionIndex" = "Answer"."userAnswer") THEN 1
                  ELSE NULL::integer
              END) AS "correctAnswerCount",
          count(
              CASE
                  WHEN ("Question"."correctOptionIndex" <> "Answer"."userAnswer") THEN 1
                  ELSE NULL::integer
              END) AS "incorrectAnswerCount",
          count(
              CASE
                  WHEN ("Answer"."userAnswer" = 0) THEN 1
                  ELSE NULL::integer
              END) AS "option1AnswerCount",
          count(
              CASE
                  WHEN ("Answer"."userAnswer" = 1) THEN 1
                  ELSE NULL::integer
              END) AS "option2AnswerCount",
          count(
              CASE
                  WHEN ("Answer"."userAnswer" = 2) THEN 1
                  ELSE NULL::integer
              END) AS "option3AnswerCount",
          count(
              CASE
                  WHEN ("Answer"."userAnswer" = 3) THEN 1
                  ELSE NULL::integer
              END) AS "option4AnswerCount",
          count(
              CASE
                  WHEN (("Answer"."incorrectAnswerReason")::text = '1'::text) THEN 1
                  ELSE NULL::integer
              END) AS "incorrectReason1Count",
          count(
              CASE
                  WHEN (("Answer"."incorrectAnswerReason")::text = '2'::text) THEN 1
                  ELSE NULL::integer
              END) AS "incorrectReason2Count",
          count(
              CASE
                  WHEN (("Answer"."incorrectAnswerReason")::text = '3'::text) THEN 1
                  ELSE NULL::integer
              END) AS "incorrectReason3Count"
          FROM (public."Question"
            JOIN public."Answer" ON (
                ("Question".id = "Answer"."questionId") AND 
                ("Question".type = ANY (ARRAY['MCQ-SO'::public."enum_Question_type", 'MCQ-AR'::public."enum_Question_type"]))
              )
            )
          WHERE
            -- added this constraint to utilize partial index on `Answer`.`createdAt`
            "Answer"."createdAt" >= '2021-04-01 00:00:00+00'::TIMESTAMPTZ AND 
            "Answer"."createdAt"::TIMESTAMPTZ >= dateLower AND "Answer"."createdAt"::TIMESTAMPTZ <= dateUpper AND
            CASE 
              WHEN questionId IS NULL THEN TRUE 
              ELSE "Question".id = questionId 
            END
        GROUP BY "Question".id ) t ON CONFLICT (id) DO UPDATE SET
        -- need to recompute correctPercentage & difficultyLevel
          "correctPercentage" = get_correct_percentage("QuestionAnalyticsTable"."correctAnswerCount" + EXCLUDED."correctAnswerCount", "QuestionAnalyticsTable"."incorrectAnswerCount" + EXCLUDED."incorrectAnswerCount"),
          "difficultyLevel" = get_difficulty_level("QuestionAnalyticsTable"."correctAnswerCount" + EXCLUDED."correctAnswerCount", "QuestionAnalyticsTable"."incorrectAnswerCount" + EXCLUDED."incorrectAnswerCount"),
          "correctAnswerCount" = EXCLUDED."correctAnswerCount" + "QuestionAnalyticsTable"."correctAnswerCount",
          "incorrectAnswerCount" = EXCLUDED."incorrectAnswerCount" + "QuestionAnalyticsTable"."incorrectAnswerCount",
          "option1AnswerCount" = EXCLUDED."option1AnswerCount" + "QuestionAnalyticsTable"."option1AnswerCount",
          "option2AnswerCount" = EXCLUDED."option2AnswerCount" + "QuestionAnalyticsTable"."option2AnswerCount",
          "option3AnswerCount" = EXCLUDED."option3AnswerCount" + "QuestionAnalyticsTable"."option3AnswerCount",
          "option4AnswerCount" = EXCLUDED."option4AnswerCount" + "QuestionAnalyticsTable"."option4AnswerCount",
          "incorrectReason1Count" = EXCLUDED."incorrectReason1Count" + "QuestionAnalyticsTable"."incorrectReason1Count",
          "incorrectReason2Count" = EXCLUDED."incorrectReason2Count" +  "QuestionAnalyticsTable"."incorrectReason2Count",
          "incorrectReason3Count" = EXCLUDED."incorrectReason3Count" +  "QuestionAnalyticsTable"."incorrectReason3Count";
      END
$$;

alter function refresh_question_analytics_table(timestamp with time zone, timestamp with time zone, integer) owner to learner;
