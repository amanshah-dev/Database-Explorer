CREATE FUNCTION get_practice_session_info(
    userid INTEGER, 
    testid INTEGER DEFAULT NULL::INTEGER, 
    testattemptid INTEGER DEFAULT NULL::INTEGER
) 
RETURNS JSON
LANGUAGE plpgsql
AS
$$
DECLARE
    -- Declare variables to store query results
    test_attempt_row "TestAttempt";
    test_attempt_info JSON;
    overall_performance JSON;
    difficulty_levels JSON;
    weak_topics JSON;
    streak INT;

    -- Declare additional variables for calculations
    total_time_taken INTEGER;
    total_questions INTEGER;
    average_time_per_question INTEGER;
    test_attempt_date TIMESTAMP;
    test_name TEXT;
    correct_questions INTEGER;
    incorrect_questions INTEGER;
    accuracy_rate FLOAT;
BEGIN
    -- Retrieve the TestAttempt record
    SELECT * INTO test_attempt_row
    FROM assumed_test_attempt_with_answers(userId, testId, testAttemptId);

    -- Retrieve test attempt information
    SELECT json_agg(row_to_json(t)) INTO test_attempt_info
    FROM (
        SELECT * 
        FROM get_test_attempt_info(test_attempt_row)
    ) AS t;

    -- Extracting values from test_attempt_info
    IF test_attempt_info IS NOT NULL THEN
        test_name := (test_attempt_info->0->>'test_name');
        total_time_taken := (test_attempt_info->0->>'total_time_taken');
        total_questions := (test_attempt_info->0->>'total_questions');
        
        -- Calculate average time per question
        average_time_per_question := CASE 
            WHEN total_questions > 0 THEN total_time_taken / total_questions
            ELSE 0 
        END;
        
        test_attempt_date := (test_attempt_info->0->>'test_attempt_date')::TIMESTAMP;
    ELSE
        test_name := '';
        total_time_taken := 0;
        total_questions := 0;
        average_time_per_question := 0;
    END IF;

    -- Retrieve overall performance summary of the test attempt
    SELECT json_agg(row_to_json(t)) INTO overall_performance
    FROM (
        SELECT *
        FROM get_test_attempt_summary(test_attempt_row)
    ) AS t;

    -- Extracting values from overall_performance
    IF overall_performance IS NOT NULL THEN
        correct_questions := COALESCE((overall_performance->0->>'correct_questions')::INTEGER, 0);
        incorrect_questions := COALESCE((overall_performance->0->>'incorrect_questions')::INTEGER, 0);
        accuracy_rate := COALESCE((overall_performance->0->>'accuracy_rate')::FLOAT, 0::FLOAT);
    ELSE
        correct_questions := 0;
        incorrect_questions := 0;
        accuracy_rate := 0::FLOAT;
    END IF;

    -- Retrieve correct percentage by difficulty level for the test attempt
    SELECT json_agg(
        json_build_object(
            'topic', t."difficulty_level",  -- Renaming "difficulty_level" to "topic"
            'correctPercentage', t."correct_percentage"
        )
    ) INTO difficulty_levels
    FROM get_correct_percentage_by_difficulty(test_attempt_row) AS t;

    -- Retrieve weak topics for the user based on correct percentage
    SELECT json_agg(row_to_json(t)) INTO weak_topics
    FROM (
        SELECT "topic_name" AS "weak_topic"
        FROM get_chapter_percentages(test_attempt_row)
        WHERE "correct_percentage" < 50
    ) AS t;

    -- Retrieve activity streak for the user
    SELECT get_activity_streak(userId) INTO streak;

    -- Construct the final JSON response
    RETURN jsonb_build_object(
        'components', jsonb_build_array(
            jsonb_build_object(
                'componentName', 'PracticeSessionCard',
                'props', jsonb_build_object(
                    'sessionNumber', 10, -- Assuming sessionNumber is calculated separately
                    'streak', COALESCE(streak, 0),
                    'sessionName', test_name,
                    'duration', total_time_taken,
                    'questions', total_questions
                )
            ),
            jsonb_build_object(
                'componentName', 'OverallPerformanceComponent',
                'props', jsonb_build_object(
                    'correct', correct_questions,
                    'incorrect', incorrect_questions,
                    'totalQuestions', total_questions,
                    'accuracyRate', accuracy_rate
                )
            ),
            jsonb_build_object(
                'componentName', 'TimeManagementComponent',
                'props', jsonb_build_object(
                    'averageTimePerQuestion', average_time_per_question
                )
            ),
            jsonb_build_object(
                'componentName', 'DifficultyLevelComponent',
                'props', jsonb_build_object(
                    'difficultyLevels', difficulty_levels
                )
            ),
            jsonb_build_object(
                'componentName', 'RecommendationComponent',
                'props', jsonb_build_object(
                    'weakTopics', COALESCE(weak_topics, '[]'::json)
                )
            )
        )
    );
END;
$$;

ALTER FUNCTION get_practice_session_info(integer, integer, integer) OWNER TO learner;
