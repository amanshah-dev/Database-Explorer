create function test_attempt_history_update() returns trigger
    language plpgsql
as
$$
DECLARE 
  changedAnswers jsonb := NEW."userAnswers"::jsonb - OLD."userAnswers"::jsonb;
BEGIN
  IF changedAnswers::text != '{}' THEN
    INSERT INTO "TestAttemptHistory"("eventTime", "userId", "testId", "changedAnswers")
      VALUES(CURRENT_TIMESTAMP, NEW."userId", NEW."testId", changedAnswers);
  END IF;
  RETURN NEW;
END;
$$;

alter function test_attempt_history_update() owner to learner;
