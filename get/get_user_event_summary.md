CREATE FUNCTION get_user_event_summary(user_id integer, start_date date, end_date date)
    RETURNS jsonb
    LANGUAGE plpgsql
AS
$$
BEGIN
  RETURN (
    WITH date_series AS (
      /* Generate all dates between start_date and end_date */
      SELECT generate_series(start_date::date, end_date::date, interval '1 day')::date AS date
    ),
    event_summary AS (
      /* Summarize the event counts for each date within the specified range */
      SELECT
        "eventDate"::DATE AS date,
        SUM(CASE WHEN "event" = 'Question' THEN "eventCount" ELSE 0 END) AS questions,
        SUM(CASE WHEN "event" = 'CorrectQuestion' THEN "eventCount" ELSE 0 END) AS correct,
        SUM(CASE WHEN "event" = 'PhysicsQuestion' THEN "eventCount" ELSE 0 END) AS physics_questions,
        SUM(CASE WHEN "event" = 'PhysicsCorrectQuestion' THEN "eventCount" ELSE 0 END) AS physics_correct,
        SUM(CASE WHEN "event" = 'ChemistryQuestion' THEN "eventCount" ELSE 0 END) AS chemistry_questions,
        SUM(CASE WHEN "event" = 'ChemistryCorrectQuestion' THEN "eventCount" ELSE 0 END) AS chemistry_correct
      FROM 
        "DailyUserEvent"
      WHERE 
        "userId" = user_id
        AND "eventDate" BETWEEN start_date AND end_date
      GROUP BY
        "eventDate"
    )
    SELECT jsonb_object_agg(
        ds.date::TEXT, 
        jsonb_build_object(
          'questions', COALESCE(es.questions, 0),
          'correct', COALESCE(es.correct, 0),
          'incorrect', GREATEST(COALESCE(es.questions, 0) - COALESCE(es.correct, 0), 0),
          'physicsCorrect', COALESCE(es.physics_correct, 0),
          'physicsIncorrect', GREATEST(COALESCE(es.physics_questions, 0) - COALESCE(es.physics_correct, 0), 0),
          'chemistryCorrect', COALESCE(es.chemistry_correct, 0),
          'chemistryIncorrect', GREATEST(COALESCE(es.chemistry_questions, 0) - COALESCE(es.chemistry_correct, 0), 0)
        )
      ) AS result
    FROM
      date_series ds
    LEFT JOIN event_summary es ON ds.date = es.date
  );
END;
$$;

ALTER FUNCTION get_user_event_summary(integer, date, date) OWNER TO neetprep_rw;
