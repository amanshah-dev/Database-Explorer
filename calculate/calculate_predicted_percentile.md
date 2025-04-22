-- Create function to calculate predicted percentile based on user answers and correctness
create function calculate_predicted_percentile(test_attempt_id integer) returns jsonb
    language plpgsql
as
$$
DECLARE
    user_answers JSONB;
    question_data JSONB;
    question_id INT;
    user_option TEXT;
    correct_option INT;
    correctness_ratio FLOAT;
    user_answer BOOLEAN;
    low FLOAT := 0;
    high FLOAT := 100;
    spread FLOAT;
    mid FLOAT;
    undamped_d FLOAT;
    damping FLOAT;
    d FLOAT;
    range_factor FLOAT;
    d1 FLOAT;
    d2 FLOAT;
    result JSONB;
    correct_answers JSONB := '[]'::JSONB;
    incorrect_answers JSONB := '[]'::JSONB;
    test_questions JSONB;
    all_questions JSONB;
BEGIN
    -- Fetch user answers for the given test attempt
    SELECT "userAnswers" INTO user_answers
    FROM "TestAttempt"
    WHERE "id" = test_attempt_id;

    -- Fetch all questions for the test
    SELECT array_to_json(array_agg(jsonb_build_object('question_id', "Question".id, 'correctOptionIndex', "correctOptionIndex"))) INTO test_questions
    FROM "TestQuestion"
    JOIN "Question" ON "TestQuestion"."questionId" = "Question".id
    WHERE "TestQuestion"."testId" = (SELECT "testId" FROM "TestAttempt" WHERE id = test_attempt_id);

    -- Loop through all test questions
    FOR question_data IN SELECT * FROM jsonb_array_elements(test_questions)
    LOOP
        question_id := (question_data->>'question_id')::INT;
        correct_option := (question_data->>'correctOptionIndex')::INT;

        -- Check if the question was attempted
        IF user_answers ? question_id::TEXT THEN
            user_option := trim(both '"' from (user_answers->>question_id::TEXT)::TEXT);
            user_answer := (user_option::INT = correct_option);
        ELSE
            user_answer := FALSE;
        END IF;

        -- Assume correctnessRatio to be 0.5 if not available
        SELECT COALESCE((SELECT "correctPercentage" FROM "QuestionAnalytics" WHERE "id" = question_id), 50)
        INTO correctness_ratio;

        correctness_ratio := correctness_ratio / 100;

        -- Separate into correct and incorrect answers
        IF user_answer THEN
            correct_answers := correct_answers || jsonb_build_object('question_id', question_id, 'correctness_ratio', correctness_ratio, 'user_answer', user_answer);
        ELSE
            incorrect_answers := incorrect_answers || jsonb_build_object('question_id', question_id, 'correctness_ratio', correctness_ratio, 'user_answer', user_answer);
        END IF;
    END LOOP;

    -- Sort correct answers in decreasing order of correctness_ratio
    SELECT jsonb_agg(data ORDER BY (data->>'correctness_ratio')::FLOAT DESC)
    INTO correct_answers
    FROM jsonb_array_elements(correct_answers) AS data;

    -- Sort incorrect answers in increasing order of correctness_ratio
    SELECT jsonb_agg(data ORDER BY (data->>'correctness_ratio')::FLOAT)
    INTO incorrect_answers
    FROM jsonb_array_elements(incorrect_answers) AS data;

    -- Loop over sorted correct answers
    FOR question_data IN SELECT * FROM jsonb_array_elements(correct_answers)
    LOOP
        question_id := (question_data->>'question_id')::INT;
        correctness_ratio := (question_data->>'correctness_ratio')::FLOAT;
        user_answer := (question_data->>'user_answer')::BOOLEAN;

        -- Calculate d
        spread := high - low;
        mid := spread * (1 - correctness_ratio);
        undamped_d := CASE WHEN user_answer THEN mid - low ELSE high - mid END;
        damping := 1 - mid / 100;
        d := undamped_d * damping;

        -- Calculate range factor
        range_factor := 0.005 * (spread + 100);

        -- Split d into d1 and d2
        d1 := d * range_factor;
        d2 := d - d1;

        -- Update low and high based on user answer
        IF user_answer THEN
            low := low + d1;
            high := LEAST(100, high + d2);
        ELSE
            low := GREATEST(0, low - d2);
            high := high - d1;
        END IF;
    END LOOP;

    -- Loop over sorted incorrect answers
    FOR question_data IN SELECT * FROM jsonb_array_elements(incorrect_answers)
    LOOP
        question_id := (question_data->>'question_id')::INT;
        correctness_ratio := (question_data->>'correctness_ratio')::FLOAT;
        user_answer := (question_data->>'user_answer')::BOOLEAN;

        -- Calculate d
        spread := high - low;
        mid := spread * (1 - correctness_ratio);
        undamped_d := CASE WHEN user_answer THEN mid - low ELSE high - mid END;
        damping := 1 - mid / 100;
        d := undamped_d * damping;

        -- Calculate range factor
        range_factor := 0.005 * (spread + 100);

        -- Split d into d1 and d2
        d1 := d * range_factor;
        d2 := d - d1;

        -- Update low and high based on user answer
        IF user_answer THEN
            low := low + d1;
            high := LEAST(100, high + d2);
        ELSE
            low := GREATEST(0, low - d2);
            high := high - d1;
        END IF;
    END LOOP;

    -- Calculate final percentile and spread
    spread := (high - low) / 2;
    result := jsonb_build_object(
        'percentile', low + spread,
        'spread', spread
    );

    RETURN result;
END;
$$;

-- Alter function owner
alter function calculate_predicted_percentile(integer) owner to neetprep_rw;
